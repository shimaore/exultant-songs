Auto-call
---------

This module places calls (presumably towards a client or Centrex extension) and sends them into the socket we control.
It ensures data is retrieved and injected in the call.

    seem = require 'seem'
    FS = require 'esl'

    pkg = require '../package'
    @name = "#{pkg.name}:place-call"
    debug = (require 'debug') @name

    @notify = ({cfg,socket}) ->

* cfg.session.profile (string) Sofia profile that should be used to place calls towards client, for automated calls.

      unless @cfg.session?.profile?
        debug 'Missing cfg.session.profile, not starting.'
        return

      unless @cfg.update_session_reference_data?
        debug 'Missing cfg.update_session_reference_data, not starting.'
        return

See conf/freeswitch

      socket_port = @cfg.profiles?[@cfg.session.profile]?.socket_port ? 5721

Connect a single client, and push new calls through it. The calls are automatically sent back to ourselves so that they can be processed like regular outbound calls.

`data` should contain:
- `_id`
- `account`
- `endpoint`

      client = FS.client ->
        socket.on 'place-call', seem (data) =>
          return unless data._id?.match /^[\w-]+$/
          debug 'Placing call towards caller'

FIXME The data sender must do resolution of the endpoint_via and associated translations????

          yield cfg.update_session_reference_data data
          data._in = [
            "endpoint:#{data.endpoint}"
            "account:#{data.account}"
          ]
          data.host = @cfg.host
          data.state = 'connecting-caller'
          socket.emit 'reference', data
          {uuid} = res = yield @api "originate {session_reference=#{data._id},origination_uuid=#{data._id}}sofia/#{@cfg.session.profile}-egress/sip:#{data.caller} &socket(127.0.0.1:#{socket_port} async full)"
          data.state = 'caller-connected'
          data.originate_uuid = uuid
          socket.emit 'reference', data
          yield cfg.update_session_reference_data data
          debug 'Session state:', data.state
        debug 'Client ready'

      client.connect (@cfg.socket_port ? 5722), '127.0.0.1'

      socket.emit 'configure', dial_calls: true
      debug 'Module Ready'
