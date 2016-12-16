Register the database we use to store session state.
FIXME: use redis instead.

    seem = require 'seem'
    PouchDB = require 'pouchdb'
    uuidV4 = require 'uuid/v4'
    debug = (require 'debug') 'exultant-songs:middleware:reference_in_pouchdb'

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    @server_pre = ->

* cfg.session.db (URI) database used to store automated / complete call records (call-center oriented)

      unless @cfg.session?.db?
        debug 'Missing cfg.session.db, not starting.'
        return

* cfg.session.db (URI) The PouchDB URI of the database used to store call reference data

      db = new PouchDB @cfg.session.db

      @cfg.get_session_reference_data = get_data = (id) ->
        id ?= new uuidV4()
        db
          .get id
          .catch -> _id:id

      @cfg.update_session_reference_data = save_data = seem (data,tries = 3) ->
        prev = yield get_data data._id
        for own k,v of data when k[0] isnt '_'
          prev[k] = v
        {rev} = yield db
          .put data
          .catch seem (error) ->
            debug "error: #{error.stack ? error}"
            if tries-- > 0
              yield sleep 173
              save_data data
            rev: data._rev
        data._rev = rev