    seem = require 'seem'
    describe 'Modules', ->
      list = [
          'middleware/place-call.coffee.md'
        ]

      unit = (m) ->
        it "should load #{m}", seem ->
          ctx =
            cfg:
              sip_profiles:{}
              prefix_admin: ''
            session:{}
            call:
              once: -> Promise.resolve null
              linger: -> Promise.resolve null
            req:
              variable: -> null
            data:
              'Channel-Context': 'sbc-ingress'
          M = require "../#{m}"
          yield M.notify.call ctx, ctx

      for m in list
        unit m
