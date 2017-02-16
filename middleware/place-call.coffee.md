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
          debug 'received place-call', data
          return unless data._id?.match /^[\w-]+$/
          debug 'Placing call towards caller'

FIXME The data sender must do resolution of the endpoint_via and associated translations????
ANSWER: Yes. And store the result in `caller`.

          yield cfg.update_session_reference_data data
          data._in = [
            "endpoint:#{data.endpoint}"
            "account:#{data.account}"
          ]
          data.host = cfg.host
          data.state = 'connecting-caller'
          socket.emit 'reference', data

          profile = "huge-play-#{cfg.session.profile}-egress"
          context = "#{cfg.session.profile}-egress"

          options =

These are used by `huge-play/middleware/client/setup`.

            session_reference: data._id
            origination_context: context

And `ccnq4-opensips` requires `X-CCNQ3-Endpoint` for routing.

            'sip_h_X-CCNQ3-Endpoint': data.endpoint

Finally we ensure we can track the call by forcing its UUID.

            origination_uuid: data._id

          params = ("#{k}=#{v}" for own k,v of options).join ','

          sofia = "{#{params}}sofia/#{profile}/sip:#{data.caller}"
          command = "socket(127.0.0.1:#{socket_port} async full)"
          argv = [
            sofia
            "'&#{command}'"

dialplan

            'none'

context

            context

cid_name -- callee_name, shows on the caller's phone and in Channel-(Orig-)Callee-ID-Name

            data.callee_name ? pkg.name

cid_num -- called_num, shows on the caller's phone and in Channel-(Orig-)Callee-ID-Number

            data.callee_num ? data.destination

timeout_sec

            data.call_timeout ? ''

          ].join ' '
          cmd = "originate #{argv}"

          debug "Calling #{cmd}"

          res = yield @api(cmd).catch (error) ->
            msg = error.stack ? error.toString()
            debug "originate: #{msg}"
            {error:"#{msg}"}

The `originate` command will return when the call is answered by the callee (or an error occurred).

          debug "Originate returned", res
          if res.body?[0] is '+'
            data.state = 'caller-connected'
          else
            data.state = 'caller-failed'
          socket.emit 'reference', data
          yield cfg.update_session_reference_data data
          debug 'Session state:', data.state
        debug 'Client ready'

      client.connect (@cfg.socket_port ? 5722), '127.0.0.1'

      socket.on 'configured', (data) ->
        debug 'Socket configured', data

      socket.emit 'register', event: 'place-call', default_room:'dial_calls'
      socket.emit 'configure', dial_calls: true

      debug 'Module Ready'
