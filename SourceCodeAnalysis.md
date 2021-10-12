Diagram for codes https://drive.mindmup.com/map/1uliifVE0_OkGTfW91Gw0iNZojiRqbF60


`NewTCPEntryPoints`  
Generates handles for each TCP entry points which was configured and put them in a map which key is entry name `serverEntryPointsTCP[entryPointName]`.
`NewTCPEntryPoint` builds a listener `buildListener` and create http server and https server both of which was support by standard golang net/http libraries, what's more http server will support http2. Also, it will support http3 which was based on  `quic` protocol. Then set http forwarder `rt.HTTPForwarder(httpServer.Forwarder)` and https forwarder `rt.HTTPSForwarder(httpsServer.Forwarder)`.
At last, set handle of this entry point `tcpSwitcher.Switch(rt)`

`NewConfigurationWatcher` 
Generates a `watcher` which is used to watch configuration updates so that **Traefik** could be configured dynamically. `watcher.AddListener` is provided to register handles of each dynamic configuration to handle the event of each configurations' changes.   

`switchRouter` Handle routers' change events. `routerFactory.CreateRouters(rtConf)` will create tcp routers and udp routers according the latest router configuration `rtConf` and update them `serverEntryPointsTCP.Switch(routers)` `serverEntryPointsUDP.Switch(udpRouters)`  

In `routerFactory.CreateRouters(rtConf)`  
1. `f.managerFactory.Build(rtConf)` creates a serviceManager, and start health check `serviceManager.LaunchHealthCheck()`  
2. `middleware.NewBuilder(rtConf.Middlewares, serviceManager, f.pluginBuilder)` creates a middlewaresBuilder  
3. `router.NewManager(rtConf, serviceManager, middlewaresBuilder, f.chainBuilder, f.metricsRegistry)`creates a routerManager,which creates http handler `routerManager.BuildHandlers(ctx, f.entryPointsTCP, false)` and https handlers `routerManager.BuildHandlers(ctx, f.entryPointsTCP, true)`  
Then  
4. `tcp.NewManager(rtConf)` creates a tcp serviceManager  
5. `middlewaretcp.NewBuilder(rtConf.TCPMiddlewares)` creates a tcp middlewaresBuilder   
6. `routertcp.NewManager(rtConf, svcTCPManager, middlewaresTCPBuilder, handlersNonTLS, handlersTLS, f.tlsManager)` creates a tcp routerManager,and create tcp handler `rtTCPManager.BuildHandlers(ctx, f.entryPointsTCP)`  
Last
7. `udp.NewManager(rtConf)` creates an udp serviceManager  
8. `routerudp.NewManager(rtConf, svcUDPManager)` creates an udp routerManager,and create udp handler `rtUDPManager.BuildHandlers(ctx, f.entryPointsUDP)`

`m.serviceManager.BuildHTTP(ctx, router.Service)` creates a handler for http/https which depends on the configuration `getLoadBalancerServiceHandler`,`getWRRServiceHandler`,`getMirrorServiceHandler`, actually this handler is implemented by a `httputil.ReverseProxy` which created in `buildProxy(service.PassHostHeader, service.ResponseForwarding, roundTripper, m.bufferPool)`,  
and add this handler to a router, the key of which is the configured rule and priority `router.AddRoute(routerConfig.Rule, routerConfig.Priority, handler)`

`rtTCPManager.BuildHandlers(ctx, f.entryPointsTCP)` creates tcp handler for each entryPoints. 
In `m.buildEntryPointHandler(ctx, routers, entryPointsRoutersHTTP[entryPointName], m.httpHandlers[entryPointName], m.httpsHandlers[entryPointName])`  
a tcp router will get the latest http handler `router.HTTPHandler(handlerHTTP)` and https handler `router.HTTPSHandler(sniCheck, defaultTLSConf)` both of which are created above `m.serviceManager.BuildHTTP(ctx, router.Service)`
and of course tco handler will be created here `m.buildTCPHandler(ctxRouter, routerConfig)`

So we could know that http/https handler is actually a `httputil.ReverseProxy`, and tcp handler is implemented by Traefik, this is the code:
```go
type Proxy struct {
    address          string
    target           *net.TCPAddr
    terminationDelay time.Duration
    proxyProtocol    *dynamic.ProxyProtocol
    refreshTarget    bool
}

func (p *Proxy) ServeTCP(conn WriteCloser) {
	log.WithoutContext().Debugf("Handling connection from %s", conn.RemoteAddr())

	// needed because of e.g. server.trackedConnection
	defer conn.Close()

	connBackend, err := p.dialBackend()
	if err != nil {
		log.WithoutContext().Errorf("Error while connecting to backend: %v", err)
		return
	}

	// maybe not needed, but just in case
	defer connBackend.Close()
	errChan := make(chan error)

	if p.proxyProtocol != nil && p.proxyProtocol.Version > 0 && p.proxyProtocol.Version < 3 {
		header := proxyproto.HeaderProxyFromAddrs(byte(p.proxyProtocol.Version), conn.RemoteAddr(), conn.LocalAddr())
		if _, err := header.WriteTo(connBackend); err != nil {
			log.WithoutContext().Errorf("Error while writing proxy protocol headers to backend connection: %v", err)
			return
		}
	}

	go p.connCopy(conn, connBackend, errChan)
	go p.connCopy(connBackend, conn, errChan)

	err = <-errChan
	if err != nil {
		log.WithoutContext().Errorf("Error during connection: %v", err)
	}

	<-errChan
}
```
This describes how does Traefik start to serve.  
`s.tcpEntryPoints.Start()` serves tcp requests, `s.tcpEntryPoints.Start()` serves udp requests and `s.watcher.Start()` watches rule configuration changes
```go
func (s *Server) Start(ctx context.Context) {
	go func() {
		<-ctx.Done()
		logger := log.FromContext(ctx)
		logger.Info("I have to go...")
		logger.Info("Stopping server gracefully")
		s.Stop()
	}()

	s.tcpEntryPoints.Start()
	s.udpEntryPoints.Start()
	s.watcher.Start()

	s.routinesPool.GoCtx(s.listenSignals)
}
```
When a request message comes to a tcp entryPoint, tcp listener accepts the tcp connection `conn, err := e.listener.Accept()`,  
then serve this connection `e.switcher.ServeTCP(newTrackedConnection(writeCloser, e.tracker))`. And in `(e *TCPEntryPoint) SwitchRouter(rt *tcp.Router)`, `e.switcher.Switch(rt)` has set the newest tcp router  
```go
type Router struct {
	routingTable      map[string]Handler
	httpForwarder     Handler
	httpsForwarder    Handler
	httpHandler       http.Handler
	httpsHandler      http.Handler
	httpsTLSConfig    *tls.Config // default TLS config
	catchAllNoTLS     Handler
	hostHTTPTLSConfig map[string]*tls.Config // TLS configs keyed by SNI
}
func (r *Router) ServeTCP(conn WriteCloser) {
	// FIXME -- Check if ProxyProtocol changes the first bytes of the request

if r.catchAllNoTLS != nil && len(r.routingTable) == 0 {
    r.catchAllNoTLS.ServeTCP(conn)
    return
}

br := bufio.NewReader(conn)
    serverName, tls, peeked, err := clientHelloServerName(br)
    if err != nil {
    conn.Close()
    return
}

// Remove read/write deadline and delegate this to underlying tcp server (for now only handled by HTTP Server)
err = conn.SetReadDeadline(time.Time{})
    if err != nil {
    log.WithoutContext().Errorf("Error while setting read deadline: %v", err)
}

err = conn.SetWriteDeadline(time.Time{})
if err != nil {
    log.WithoutContext().Errorf("Error while setting write deadline: %v", err)
}

if !tls {
    switch {
        case r.catchAllNoTLS != nil:
            r.catchAllNoTLS.ServeTCP(r.GetConn(conn, peeked))
        case r.httpForwarder != nil:
            r.httpForwarder.ServeTCP(r.GetConn(conn, peeked))
        default:
            conn.Close()
}
    return
}

// FIXME Optimize and test the routing table before helloServerName
serverName = types.CanonicalDomain(serverName)
if r.routingTable != nil && serverName != "" {
    if target, ok := r.routingTable[serverName]; ok {
        target.ServeTCP(r.GetConn(conn, peeked))
        return
    }
}

// FIXME Needs tests
if target, ok := r.routingTable["*"]; ok {
    target.ServeTCP(r.GetConn(conn, peeked))
    return
}

if r.httpsForwarder != nil {
    r.httpsForwarder.ServeTCP(r.GetConn(conn, peeked))
} else {
    conn.Close()
    }
}
```
In this handle function, we could see this will send connection to http if it's not nil `r.httpForwarder.ServeTCP(r.GetConn(conn, peeked))`,  
we could find `httpForwarder` is just 
```go
serverHTTP := &http.Server{
		Handler:      handler,
		ErrorLog:     httpServerLogger,
		ReadTimeout:  time.Duration(configuration.Transport.RespondingTimeouts.ReadTimeout),
		WriteTimeout: time.Duration(configuration.Transport.RespondingTimeouts.WriteTimeout),
		IdleTimeout:  time.Duration(configuration.Transport.RespondingTimeouts.IdleTimeout),
	}
```
and this actually a wrapper of ` middlewares.NewHandlerSwitcher(router.BuildDefaultHTTPRouter())` or `handler = h2c.NewHandler(handler, &http2.Server{})` if http 2.0 was chosen,  this server serves a listener `newHTTPForwarder(ln)` which has 
tcp connection as an underlying connection, however it's accept method was modified
```go
func (h *httpForwarder) Accept() (net.Conn, error) {
	select {
	case conn := <-h.connChan:
		return conn, nil
	case err := <-h.errChan:
		return nil, err
	}
}
func (h *httpForwarder) ServeTCP(conn tcp.WriteCloser) {
    h.connChan <- conn
}
```
So in above code logic `r.httpForwarder.ServeTCP(r.GetConn(conn, peeked))`, this connection will be sent to a http server as an underlying connection.