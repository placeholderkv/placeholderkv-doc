Return the UNIX TIME of the last DB save executed with success.
A client may check if a `BGSAVE` command succeeded reading the `LASTSAVE` value,
then issuing a `BGSAVE` command and checking at regular intervals every N
seconds if `LASTSAVE` changed. Valkey considers the database saved successfully at startup.
