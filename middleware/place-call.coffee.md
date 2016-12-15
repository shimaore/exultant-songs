Auto-call
---------

This module places calls (presumably towards a client or Centrex extension) and sends them into the socket we control.
It ensures data is retrieved and injected in the call.

    seem = require 'seem'
    PouchDB = require 'shimaore-pouchdb'
    FS = require 'esl'

    db = null

    pkg = require '../package'
    @name = "#{pkg.name}:place-call"
    debug = (require 'debug') @name

    @notify = ({socket}) ->

Report `place-call` messages over socket.io.

      @cfg.statistics?.on 'place-call', (data) ->
        socket.emit 'place-call', data

* cfg.session.db (URI) database used to store automated calls records.

      unless @cfg.session?.db?
        debug 'Missing cfg.session.db, not starting.'
        return

* cfg.session.profile (string) Sofia profile that should be used to place calls towards client, for automated calls.

      unless @cfg.session?.profile?
        debug 'Missing cfg.session.profile, not starting.'
        return

See conf/freeswitch

      socket_port = @cfg.profiles?[@cfg.session.profile]?.socket_port ? 5721

Register the database we use to store session state.
FIXME: use redis instead.

      db = new PouchDB @cfg.session.db

Connect a single client, and push new calls through it. The calls are automatically sent back to ourselves so that they can be processed like regular outbound calls.

      client = FS.client ->
        socket.on 'place-call', seem (data) =>
          return unless data._id?.match /^[\w-]+$/
          debug 'Placing call towards caller'
          {rev} = yield db.put data
          data._rev = rev
          data._in = [
            "endpoint:#{data.endpoint}"
            "account:#{data.account}"
          ]
          data.host = @cfg.host
          data.state = 'connecting-caller'
          socket.emit 'place-call', data
          {uuid} = res = yield @api "originate {session_reference=#{data._id}}sofia/#{@cfg.session.profile}-egress/sip:#{data.caller} &socket(127.0.0.1:#{socket_port} async full)"
          data.state = 'caller-connected'
          socket.emit 'place-call', data
          debug 'Caller connected'
        debug 'Client ready'

      client.connect (@cfg.socket_port ? 5722), '127.0.0.1'
      debug 'Module Ready'

    @include = seem ->
      session_reference = @req.variable 'session_reference'
      return unless session_reference?

      @session.placed_call_data = data = yield db.get session_reference

Inject data from the session reference as-if it had been provided to us by FreeSwitch.

- data set by the useful-wind/router

      @source ?= data.source
      @destination ?= data.destination

- data used by this module

      @call["variable_sip_h_P-Charge-Info"] ?= data.account
      @call["variable_sip_h_X-CCNQ3-Endpoint"] ?= data.endpoint

TBD: Check whether Channel-Context is pre-set or not.

      @data['Channel-Context'] = [ @cfg.session.profile, 'egress' ].join '-'

      data.state = 'routing'
      @statistics.emit 'place-call', data

Send a notification at the end of the call leg.

      @call.once 'cdr_report', (report) =>
        data.state = 'reporting'
        for own k,v of report
          data[k] ?= v
        @statistics?.emit 'place-call', data
