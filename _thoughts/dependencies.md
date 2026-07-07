No need to do dependency injection on most stuff:

logging - bad smell, logging has plenty of config to silence it or redirect it. leave it be.
prometheus metrics / telemetry - these are just counters withing the same process, why the hell would you do that
database/orm - just test with the damn thing and optimize if it gets very slow. You can switch to SQLite, in-memory options, run unsave DB without WAL syncs, etc
