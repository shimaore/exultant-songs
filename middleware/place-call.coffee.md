Auto-call
---------

This module places calls (presumably towards a client or Centrex extension) and sends them into the socket we control.
It ensures data is retrieved and injected in the call.

    FS = require 'esl'
    io = require 'socket.io-client'

    db = null

    pkg = require '../package'
    @name = "#{pkg.name}:place-call"
    debug = (require 'debug') @name

    @server_pre = ->

* cfg.notify (socket.io URI) used to receive notifications (`place-call` messages) for automated calls.

      unless @cfg.notify?
        debug 'Missing cfg.notify, not starting.'
        return

* cfg.session.db (URI) database used to store automated calls records.

      unless @cfg.session?.db?
        debug 'Missing cfg.session_db, not starting.'
        return

* cfg.session.profile (string) Sofia profile that should be used to place calls towards client, for automated calls.

      unless @cfg.session?.profile?
        debug 'Missing cfg.session.profile, not starting.'
        return

See conf/freeswitch

      socket_port = @cfg.profiles?[@cfg.session.profile]?.socket_port ? 5721

      socket = io @cfg.notify
      db = new PouchDB @cfg.session.db

Connect a single client, and push new calls through it. The calls are automatically sent back to ourselves so that they can be processed like regular outbound calls.

      client = FS.client ->
        socket.on 'place-call', (data) ->
          debug 'Placing call towards caller'
          {rev} = yield db.put data
          data._rev = rev
          socket.emit 'place-call:connecting-caller', data
          {uuid} = res = yield @api "originate {session_reference=#{data._id}}sofia/#{@cfg.session.profile}/sip:#{data.caller} &socket(127.0.0.1:#{socket_port} async full)"
          socket.emit 'place-call:caller-connected', data
          debug 'Caller connected'
        debug 'Client ready'

      client.connect (@cfg.socket_port ? 5722), '127.0.0.1'
      debug 'Module Ready'

    @include = ->
      return unless @data.session_reference?

      @session.placed_call_data = data = yield db.get @data.session_reference

Inject data from the session reference as-if it had been provided to us by FreeSwitch.

- data set by the useful-wind/router

      @source ?= data.source
      @destination ?= data.destination

- data used by this module

      @call["variable_sip_h_P-Charge-Info"] ?= data.account
      @call["vairable_sip_h_X-CCNQ3-Endpoint"] ?= data.endpoint
      @data['Channel-Context'] = [@cfg.session.profile,'egress'].join '-'
