# map-hosting-setup-hu
Magyar nyelvű leírás térkép file szerver beállításához

# Info

A beállítás ez alapján a leírás alapján történt: https://blog.astrid-guenther.de/en/maplibregl/maplibre-vector-tilesen/
Itt megpróbálom leírni a hostinghoz szükséges lényeges pontokat, kitérve a főbb hibalehetőségekre.

# Tilemaker install és használat

A tilemaker a térképek átalakításához kell.

Installálás:

>     mkdir tile
>     cd tile
>     git clone https://github.com/systemed/tilemaker.git
>     cd tilemaker
>     # Az eredeti leírásban a lua5.1 nem volt a következő parancsban, de úgy tapasztaltam, hogy kell.
>     sudo apt install build-essential libboost-dev libboost-filesystem-dev libboost-iostreams-dev libboost-program-options-dev libboost-system-dev liblua5.1-0-dev libprotobuf-dev libshp-dev libsqlite3-dev protobuf-compiler rapidjson-dev lua5.1
>     cd tilemaker
>     make
>     sudo make install

Az eredeti leírás coastlineüokat is használ, én ezt nem használtam mivel csak Magyarország térképpel próbáltam. Teljes Európa térkép generálásához kellenek a coastlineok az eredeti leírás alapján.
Használat előtt a tömörítést ki kell kapcsolni az adott konfigban, pl. tilemaker/resources/config-openmaptiles.json
Meg kell keresni a következő sort:
>     compress:"gzip"
és megváltoztatni erre:
>     compress:"none"

Használat:
./tilemaker --input $HOME/tile/hungary-latest.osm.pbf --output $HOME/tile/tilemaker/hungary --process $HOME/tile/tilemaker/resources/process-openmaptiles.lua --config $HOME/tile/tilemaker/resources/config-openmaptiles.json > $HOME/tile/tilemaker-hungary.log &

