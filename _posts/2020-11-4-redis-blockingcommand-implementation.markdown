---
layout: post
title:  "redis之阻塞命令的实现"
date:   2020-11-4 15:00:00
categories: blocking-command
tags: blocking-command
excerpt: redis之阻塞命令的实现
mathjax: true
---

#### 概述
redis的list数据结构中有一些阻塞命令，我们以brpop为例进行分析，其他的阻塞命令实现以此类推。
```shell
brpop key [key...] timeout
```
在一个redis客户端中可以对多个key进行阻塞，直到超时或者任意一个key有可弹出元素为止。同一个key也可以被多个客户端同时阻塞，按照先阻塞先服务进行元素弹出。

#### 阻塞的实现
首先我们看一下brpop命令是如何阻塞客户端的
```c
/* Blocking RPOP/LPOP */
void blockingPopGenericCommand(client *c, int where) {
    robj *o;
    mstime_t timeout;
    int j;

    if (getTimeoutFromObjectOrReply(c,c->argv[c->argc-1],&timeout,UNIT_SECONDS)
        != C_OK) return;

    for (j = 1; j < c->argc-1; j++) {
        o = lookupKeyWrite(c->db,c->argv[j]);
        if (o != NULL) {
            if (o->type != OBJ_LIST) {
                addReply(c,shared.wrongtypeerr);
                return;
            } else {
                if (listTypeLength(o) != 0) {
                    /* Non empty list, this is like a non normal [LR]POP. */
                    char *event = (where == LIST_HEAD) ? "lpop" : "rpop";
                    robj *value = listTypePop(o,where);
                    serverAssert(value != NULL);

                    addReplyArrayLen(c,2);
                    addReplyBulk(c,c->argv[j]);
                    addReplyBulk(c,value);
                    decrRefCount(value);
                    notifyKeyspaceEvent(NOTIFY_LIST,event,
                                        c->argv[j],c->db->id);
                    if (listTypeLength(o) == 0) {
                        dbDelete(c->db,c->argv[j]);
                        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",
                                            c->argv[j],c->db->id);
                    }
                    signalModifiedKey(c,c->db,c->argv[j]);
                    server.dirty++;

                    /* Replicate it as an [LR]POP instead of B[LR]POP. */
                    rewriteClientCommandVector(c,2,
                        (where == LIST_HEAD) ? shared.lpop : shared.rpop,
                        c->argv[j]);
                    return;
                }
            }
        }
    }

    /* If we are inside a MULTI/EXEC and the list is empty the only thing
     * we can do is treating it as a timeout (even with timeout 0). */
    if (c->flags & CLIENT_MULTI) {
        addReplyNullArray(c);
        return;
    }

    /* If the list is empty or the key does not exists we must block */
    blockForKeys(c,BLOCKED_LIST,c->argv + 1,c->argc - 2,timeout,NULL,NULL);
}

/* This is how the current blocking lists/sorted sets/streams work, we use
 * BLPOP as example, but the concept is the same for other list ops, sorted
 * sets and XREAD.
 * - If the user calls BLPOP and the key exists and contains a non empty list
 *   then LPOP is called instead. So BLPOP is semantically the same as LPOP
 *   if blocking is not required.
 * - If instead BLPOP is called and the key does not exists or the list is
 *   empty we need to block. In order to do so we remove the notification for
 *   new data to read in the client socket (so that we'll not serve new
 *   requests if the blocking request is not served). Also we put the client
 *   in a dictionary (db->blocking_keys) mapping keys to a list of clients
 *   blocking for this keys.
 * - If a PUSH operation against a key with blocked clients waiting is
 *   performed, we mark this key as "ready", and after the current command,
 *   MULTI/EXEC block, or script, is executed, we serve all the clients waiting
 *   for this list, from the one that blocked first, to the last, accordingly
 *   to the number of elements we have in the ready list.
 */

/* Set a client in blocking mode for the specified key (list, zset or stream),
 * with the specified timeout. The 'type' argument is BLOCKED_LIST,
 * BLOCKED_ZSET or BLOCKED_STREAM depending on the kind of operation we are
 * waiting for an empty key in order to awake the client. The client is blocked
 * for all the 'numkeys' keys as in the 'keys' argument. When we block for
 * stream keys, we also provide an array of streamID structures: clients will
 * be unblocked only when items with an ID greater or equal to the specified
 * one is appended to the stream. */
void blockForKeys(client *c, int btype, robj **keys, int numkeys, mstime_t timeout, robj *target, streamID *ids) {
    dictEntry *de;
    list *l;
    int j;

    c->bpop.timeout = timeout;
    c->bpop.target = target;

    if (target != NULL) incrRefCount(target);

    for (j = 0; j < numkeys; j++) {
        /* Allocate our bkinfo structure, associated to each key the client
         * is blocked for. */
        bkinfo *bki = zmalloc(sizeof(*bki));
        if (btype == BLOCKED_STREAM)
            bki->stream_id = ids[j];

        /* If the key already exists in the dictionary ignore it. */
        if (dictAdd(c->bpop.keys,keys[j],bki) != DICT_OK) {
            zfree(bki);
            continue;
        }
        incrRefCount(keys[j]);

        /* And in the other "side", to map keys -> clients */
        de = dictFind(c->db->blocking_keys,keys[j]);
        if (de == NULL) {
            int retval;

            /* For every key we take a list of clients blocked for it */
            l = listCreate();
            retval = dictAdd(c->db->blocking_keys,keys[j],l);
            incrRefCount(keys[j]);
            serverAssertWithInfo(c,keys[j],retval == DICT_OK);
        } else {
            l = dictGetVal(de);
        }
        listAddNodeTail(l,c);
        bki->listnode = listLast(l);
    }
    blockClient(c,btype);
}

/* Block a client for the specific operation type. Once the CLIENT_BLOCKED
 * flag is set client query buffer is not longer processed, but accumulated,
 * and will be processed when the client is unblocked. */
void blockClient(client *c, int btype) {
    c->flags |= CLIENT_BLOCKED;
    c->btype = btype;
    server.blocked_clients++;
    server.blocked_clients_by_type[btype]++;
    addClientToTimeoutTable(c);
}
```
阻塞的总体思路，我们直接翻译blockForKeys这个方法上面的注释，
1. 当用户调用了类似blpop这类的阻塞命令的时候，如果list不为空，则直接弹出元素，此时blpop的功能等同与lpop
2. 如果list为空，server将阻塞客户端，不为客户端的socket提供新的数据，并且将key和阻塞客户端放到一个dict中，blocking_keys
3. 当被阻塞的key上面发生了push操作的时候，我们将key标记为ready状态，在执行完push操作之后，依次对阻塞这个key的客户端进行服务，也就是弹出操作。

下面按照代码详细分析下阻塞的过程
1. 在blockingPopGenericCommand方法中，list中有元素则直接弹出；没有的执行blockForKeys，进行阻塞操作。
2. 创建一个dict类型的blocking_keys，key就是阻塞的key，value是value是阻塞这个key的客户端列表。
3. 执行blockClient，阻塞客户端。将客户端标志位设置为CLIENT_BLOCKED（不对阻塞客户端执行命令），并加入clients_timeout_table（为超时取消做准备）。

我们看下blocking_keys的结构

![blocking-keys](/img/blocking/blocking-keys.png)

这样一个阻塞动作就完成了。

#### 阻塞的取消
一个阻塞的客户端有三种取消方式，别的客户端对阻塞的key执行了push操作，客户端阻塞超时，客户端主动断开连接。下面我们逐个分析一下。

##### 被动取消
当别的客户端对阻塞的keys执行push操作的时候，阻塞的客户端将解除阻塞状态，将数据返回给客户端，我们看下代码
```c
/* If the specified key has clients blocked waiting for list pushes, this
 * function will put the key reference into the server.ready_keys list.
 * Note that db->ready_keys is a hash table that allows us to avoid putting
 * the same key again and again in the list in case of multiple pushes
 * made by a script or in the context of MULTI/EXEC.
 *
 * The list will be finally processed by handleClientsBlockedOnKeys() */
void signalKeyAsReady(redisDb *db, robj *key) {
    readyList *rl;

    /* No clients blocking for this key? No need to queue it. */
    if (dictFind(db->blocking_keys,key) == NULL) return;

    /* Key was already signaled? No need to queue it again. */
    if (dictFind(db->ready_keys,key) != NULL) return;

    /* Ok, we need to queue this key into server.ready_keys. */
    rl = zmalloc(sizeof(*rl));
    rl->key = key;
    rl->db = db;
    incrRefCount(key);
    listAddNodeTail(server.ready_keys,rl);

    /* We also add the key in the db->ready_keys dictionary in order
     * to avoid adding it multiple times into a list with a simple O(1)
     * check. */
    incrRefCount(key);
    serverAssert(dictAdd(db->ready_keys,key,NULL) == DICT_OK);
}

int processCommand(client *c) {
...

/* Exec the command */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }
    return C_OK;
｝

/* This function should be called by Redis every time a single command,
 * a MULTI/EXEC block, or a Lua script, terminated its execution after
 * being called by a client. It handles serving clients blocked in
 * lists, streams, and sorted sets, via a blocking commands.
 *
 * All the keys with at least one client blocked that received at least
 * one new element via some write operation are accumulated into
 * the server.ready_keys list. This function will run the list and will
 * serve clients accordingly. Note that the function will iterate again and
 * again as a result of serving BRPOPLPUSH we can have new blocking clients
 * to serve because of the PUSH side of BRPOPLPUSH.
 *
 * This function is normally "fair", that is, it will server clients
 * using a FIFO behavior. However this fairness is violated in certain
 * edge cases, that is, when we have clients blocked at the same time
 * in a sorted set and in a list, for the same key (a very odd thing to
 * do client side, indeed!). Because mismatching clients (blocking for
 * a different type compared to the current key type) are moved in the
 * other side of the linked list. However as long as the key starts to
 * be used only for a single type, like virtually any Redis application will
 * do, the function is already fair. */
void handleClientsBlockedOnKeys(void) {
    while(listLength(server.ready_keys) != 0) {
        list *l;

        /* Point server.ready_keys to a fresh list and save the current one
         * locally. This way as we run the old list we are free to call
         * signalKeyAsReady() that may push new elements in server.ready_keys
         * when handling clients blocked into BRPOPLPUSH. */
        l = server.ready_keys;
        server.ready_keys = listCreate();

        while(listLength(l) != 0) {
            listNode *ln = listFirst(l);
            readyList *rl = ln->value;

            /* First of all remove this key from db->ready_keys so that
             * we can safely call signalKeyAsReady() against this key. */
            dictDelete(rl->db->ready_keys,rl->key);

            /* Even if we are not inside call(), increment the call depth
             * in order to make sure that keys are expired against a fixed
             * reference time, and not against the wallclock time. This
             * way we can lookup an object multiple times (BRPOPLPUSH does
             * that) without the risk of it being freed in the second
             * lookup, invalidating the first one.
             * See https://github.com/antirez/redis/pull/6554. */
            server.fixed_time_expire++;
            updateCachedTime(0);

            /* Serve clients blocked on list key. */
            robj *o = lookupKeyWrite(rl->db,rl->key);

            if (o != NULL) {
                if (o->type == OBJ_LIST)
                    serveClientsBlockedOnListKey(o,rl);
                else if (o->type == OBJ_ZSET)
                    serveClientsBlockedOnSortedSetKey(o,rl);
                else if (o->type == OBJ_STREAM)
                    serveClientsBlockedOnStreamKey(o,rl);
                /* We want to serve clients blocked on module keys
                 * regardless of the object type: we don't know what the
                 * module is trying to accomplish right now. */
                serveClientsBlockedOnKeyByModule(rl);
            }
            server.fixed_time_expire--;

            /* Free this item. */
            decrRefCount(rl->key);
            zfree(rl);
            listDelNode(l,ln);
        }
        listRelease(l); /* We have the new list on place at this point. */
    }
}

/* Helper function for handleClientsBlockedOnKeys(). This function is called
 * when there may be clients blocked on a list key, and there may be new
 * data to fetch (the key is ready). */
void serveClientsBlockedOnListKey(robj *o, readyList *rl) {
    /* We serve clients in the same order they blocked for
     * this key, from the first blocked to the last. */
    dictEntry *de = dictFind(rl->db->blocking_keys,rl->key);
    if (de) {
        list *clients = dictGetVal(de);
        int numclients = listLength(clients);

        while(numclients--) {
            listNode *clientnode = listFirst(clients);
            client *receiver = clientnode->value;

            if (receiver->btype != BLOCKED_LIST) {
                /* Put at the tail, so that at the next call
                 * we'll not run into it again. */
                listRotateHeadToTail(clients);
                continue;
            }

            robj *dstkey = receiver->bpop.target;
            int where = (receiver->lastcmd &&
                         receiver->lastcmd->proc == blpopCommand) ?
                         LIST_HEAD : LIST_TAIL;
            robj *value = listTypePop(o,where);

            if (value) {
                /* Protect receiver->bpop.target, that will be
                 * freed by the next unblockClient()
                 * call. */
                if (dstkey) incrRefCount(dstkey);
                unblockClient(receiver);

                if (serveClientBlockedOnList(receiver,
                    rl->key,dstkey,rl->db,value,
                    where) == C_ERR)
                {
                    /* If we failed serving the client we need
                     * to also undo the POP operation. */
                    listTypePush(o,value,where);
                }

                if (dstkey) decrRefCount(dstkey);
                decrRefCount(value);
            } else {
                break;
            }
        }
    }

    if (listTypeLength(o) == 0) {
        dbDelete(rl->db,rl->key);
        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",rl->key,rl->db->id);
    }
    /* We don't call signalModifiedKey() as it was already called
     * when an element was pushed on the list. */
}

void unblockClient(client *c) {
    if (c->btype == BLOCKED_LIST ||
        c->btype == BLOCKED_ZSET ||
        c->btype == BLOCKED_STREAM) {
        unblockClientWaitingData(c);
    } else if (c->btype == BLOCKED_WAIT) {
        unblockClientWaitingReplicas(c);
    } else if (c->btype == BLOCKED_MODULE) {
        if (moduleClientIsBlockedOnKeys(c)) unblockClientWaitingData(c);
        unblockClientFromModule(c);
    } else {
        serverPanic("Unknown btype in unblockClient().");
    }
    /* Clear the flags, and put the client in the unblocked list so that
     * we'll process new commands in its query buffer ASAP. */
    server.blocked_clients--;
    server.blocked_clients_by_type[c->btype]--;
    c->flags &= ~CLIENT_BLOCKED;
    c->btype = BLOCKED_NONE;
    removeClientFromTimeoutTable(c);
    queueClientForReprocessing(c);
}
```
上面的代码比较长，主要是什么时候如何解除一个阻塞的客户端
1. 在对一个key执行push操作的时候，会同时将这个key放到ready_keys里面。
2. 在执行一个客户端命令的时候，如果发现ready_keys不为空，则认为需要处理那些被阻塞的客户端
3. handleClientsBlockedOnKeys是对整个ready_keys进行遍历处理
4. serveClientsBlockedOnListKey 是对单个key所有阻塞客户端的处理，将元素从list中弹出，并且解除客户端的阻塞状态
5. 接触客户单的阻塞状态包括将客户端的标志位置位，从clients_timeout_table 这个中移出，将客户端加入到未阻塞的客户端（unblocked_clients）中，以待下次直接执行。

##### 超时取消
在redis的事件循环中的beforeSleep方法中处理了超时的阻塞客户端，
```c
/* This function is called in beforeSleep() in order to unblock clients
 * that are waiting in blocking operations with a timeout set. */
void handleBlockedClientsTimeout(void) {
    if (raxSize(server.clients_timeout_table) == 0) return;
    uint64_t now = mstime();
    raxIterator ri;
    raxStart(&ri,server.clients_timeout_table);
    raxSeek(&ri,"^",NULL,0);

    while(raxNext(&ri)) {
        uint64_t timeout;
        client *c;
        decodeTimeoutKey(ri.key,&timeout,&c);
        if (timeout >= now) break; /* All the timeouts are in the future. */
        c->flags &= ~CLIENT_IN_TO_TABLE;
        checkBlockedClientTimeout(c,now);
        raxRemove(server.clients_timeout_table,ri.key,ri.key_len,NULL);
        raxSeek(&ri,"^",NULL,0);
    }
}
```
##### 主动取消
当然，客户端也可以主动断开连接，这样也会解除客户端的阻塞状态。

#### 参考文章
1. https://www.cnblogs.com/zpcoding/p/12980362.html
2. https://www.jianshu.com/p/xsMzfn