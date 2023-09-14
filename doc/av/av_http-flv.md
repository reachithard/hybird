# 流媒体协议之Http-flv

[TOC]

## 技术原理

HTTP 协议中有个约定：content-length 字段，http 的 body 部分的长度

服务器回复http 请求的时候如果有这个字段，客户端就接收这个长度的数据然后就认 为数据传输完成了，如果服务器回复http 请求中没有这个字段，客户端就一直接收数 据，直到服务器跟客户端的 socket 连接断开。

http-flv 直播就是利用这个原理，服务器回复客户端请求的时候不加 content-length 字 段，而是携带**Transfer-Encoding: chunked**字段，在回复了 http 内容之后，紧接着发送 flv 数据，客户端就一直接收数据了，直到http连接断开。

由上可见，http-flv只适合拉流，不适合推流。

## 推流流程

* 采集数据

* 采集到的数据硬编码为H264数据

* 把编码的数据通过 FFmpeg 封装成 FLV tag

* 搭建 HTTP 服务器监听 HTTP 连接，连接成功之后发送数据

## SRS源码分析

注：推流时可能出现tcp -138错误码，需要关闭防火墙`systemctl stop firewalld`

* http初始化

```
SrsServer::SrsServer()
{
	...
    http_listener_ = new SrsTcpListener(this); // 这里会new一个dummy协程 在listen时会创建实际协程 然后调用accept， on_tcp_client
    ...
}

srs_error_t SrsServer::listen()
{
	...
    http_listener_->set_endpoint(_srs_config->get_http_stream_listen())->set_label("HTTP-Server");
    if ((err = http_listener_->listen()) != srs_success) { // 在listen时会创建实际协程 然后调用accept， on_tcp_client
    	return srs_error_wrap(err, "http server listen");
    }
    ...
}

srs_error_t SrsServer::do_on_tcp_client(ISrsListener* listener, srs_netfd_t& stfd)
{
	...
    if (listener == http_listener_ || listener == https_listener_) {
        bool is_https = listener == https_listener_;
        resource = new SrsHttpxConn(is_https, this, new SrsTcpConnection(stfd2), http_server, ip, port);
    }
    ...
}


```

* rtmp推流[这个待之后分析]

```
srs_error_t SrsRtmpConn::publishing(SrsLiveSource* source)
{
	...
	// 这里会去触发on_publish
    if ((err = acquire_publish(source)) == srs_success) {
        SrsPublishRecvThread rtrd(rtmp, req, srs_netfd_fileno(stfd), 0, this, source, _srs_context->get_id());
        err = do_publishing(source, &rtrd); // 实际推流
        rtrd.stop();
    }
    ...
    return err;
}

srs_error_t SrsRtmpConn::acquire_publish(SrsLiveSource* source)
{
    srs_error_t err = srs_success;
    ...
    err = source->on_publish();
    ...
    return err;
}

srs_error_t SrsLiveSource::on_publish()
{
    srs_error_t err = srs_success;
    ...
    if ((err = handler->on_publish(this, req)) != srs_success) {
        return srs_error_wrap(err, "handle publish");
    }
	...
    return err;
}

srs_error_t SrsServer::on_publish(SrsLiveSource* s, SrsRequest* r)
{
    srs_error_t err = srs_success;
	...
    if ((err = http_server->http_mount(s, r)) != srs_success) { // 主要是这个函数 会创建new SrsLiveStream(s, r, entry->cache); 然后注册到路由里面
        return srs_error_wrap(err, "http mount");
    }
    ...
    return err;
}

srs_error_t SrsHttpServer::http_mount(SrsLiveSource* s, SrsRequest* r)
{
    return http_stream->http_mount(s, r);
}

srs_error_t SrsHttpStreamServer::http_mount(SrsLiveSource* s, SrsRequest* r)
{
	...
    if (sflvs.find(sid) == sflvs.end()) {        
        entry = new SrsLiveEntry(mount);

        entry->source = s;
        entry->req = r->copy()->as_http();
        entry->cache = new SrsBufferCache(s, r);
        entry->stream = new SrsLiveStream(s, r, entry->cache); // 这里new了一个live stream
        
        // TODO: FIXME: maybe refine the logic of http remux service.
        // if user push streams followed:
        //     rtmp://test.com/live/stream1
        //     rtmp://test.com/live/stream2
        // and they will using the same template, such as: [vhost]/[app]/[stream].flv
        // so, need to free last request object, otherwise, it will cause memory leak.
        srs_freep(tmpl->req);
        
        tmpl->source = s;
        tmpl->req = r->copy()->as_http();
        
        sflvs[sid] = entry;
        
        // mount the http flv stream.
        // we must register the handler, then start the thread,
        // for the thread will cause thread switch context.
        if ((err = mux.handle(mount, entry->stream)) != srs_success) { // 这里进行了路由注册
            return srs_error_wrap(err, "http: mount flv stream for vhost=%s failed", sid.c_str());
        }
        
    }
    ...
    return err;
}
```

* http-flv拉流，这个在第一步的do_on_tcp_client 会new一个SrsHttpConn，里面有一个协程，然后触发cycle，

```
srs_error_t SrsHttpConn::cycle()
{
    srs_error_t err = do_cycle();
}

srs_error_t SrsHttpConn::do_cycle()
{
    srs_error_t err = srs_success;
    
    ...
    err = process_requests(&last_req);
    ...
    
    return err;
}

srs_error_t SrsHttpConn::process_request(ISrsHttpResponseWriter* w, ISrsHttpMessage* r, int rid)
{
    srs_error_t err = srs_success;
    ...
    if ((err = cors->serve_http(w, r)) != srs_success) { // 跨域
        return srs_error_wrap(err, "cors serve");
    }
    ...
    return err;
}

srs_error_t SrsHttpCorsMux::serve_http(ISrsHttpResponseWriter* w, ISrsHttpMessage* r)
{
	...
    return next_->serve_http(w, r); // next为auth 以上代码是为了跨域
}

srs_error_t SrsHttpAuthMux::serve_http(ISrsHttpResponseWriter* w, ISrsHttpMessage* r)
{
	...
    return next_->serve_http(w, r); // next为http_server 以上代码是为了auth
}

srs_error_t SrsHttpServer::serve_http(ISrsHttpResponseWriter* w, ISrsHttpMessage* r)
{
    srs_error_t err = srs_success;
    ...
    // Try http stream first, then http static if not found.
    ISrsHttpHandler* h = NULL;
    if ((err = http_stream->mux.find_handler(r, &h)) != srs_success) { // 这里会去找hander 在SrsHttpStreamServer::http_mount里面注册的
        return srs_error_wrap(err, "find handler");
    }
    if (!h->is_not_found()) {
        return http_stream->mux.serve_http(w, r);
    }
	...
    // Use http static as default server.
    return http_static->mux.serve_http(w, r);
}

srs_error_t SrsHttpServeMux::serve_http(ISrsHttpResponseWriter* w, ISrsHttpMessage* r)
{
    srs_error_t err = srs_success;
    
    ISrsHttpHandler* h = NULL;
    if ((err = find_handler(r, &h)) != srs_success) { // 再找一次
        return srs_error_wrap(err, "find handler");
    }
    
    srs_assert(h);
    if ((err = h->serve_http(w, r)) != srs_success) { // 触发实际处理函数
        return srs_error_wrap(err, "serve http");
    }
    
    return err;
}

srs_error_t SrsLiveStream::serve_http(ISrsHttpResponseWriter* w, ISrsHttpMessage* r)
{
    srs_error_t err = srs_success;
    ...
    err = do_serve_http(w, r); // 实际处理
    ...
    return err;
}

srs_error_t SrsLiveStream::do_serve_http(ISrsHttpResponseWriter* w, ISrsHttpMessage* r)
{
	...
	SrsLiveConsumer* consumer = NULL;
    SrsAutoFree(SrsLiveConsumer, consumer);
    if ((err = source->create_consumer(consumer)) != srs_success) {
        return srs_error_wrap(err, "create consumer");
    }
    if ((err = source->consumer_dumps(consumer, true, true, !enc->has_cache())) != srs_success) {
        return srs_error_wrap(err, "dumps consumer");
    }
    while (entry->enabled) {
		// 实际给客户端的数据 主要是consumer拿数据
		int count = 0;
        if ((err = consumer->dump_packets(&msgs, count)) != srs_success) {
            return srs_error_wrap(err, "consumer dump packets");
        }
        
        if (ffe) {
            err = ffe->write_tags(msgs.msgs, count); // 这里有对flv进行封装
        } else {
            err = streaming_send_messages(enc, msgs.msgs, count);
        }

    }
	...
    return srs_error_new(ERROR_HTTP_STREAM_EOF, "Stream EOF");
}
```

以上即为srs的http-flv拉流过程源码，RTMP推流目前没进行分析。待之后篇章进行补充。

// TODO 增加抓包