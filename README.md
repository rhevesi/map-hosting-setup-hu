# map-hosting-setup-hu
Magyar nyelvű leírás térkép file szerver beállításához

# Info

A beállítás ez alapján a leírás alapján történt: https://blog.astrid-guenther.de/en/maplibregl/maplibre-vector-tilesen/
Itt megpróbálom leírni a hostinghoz szükséges lényeges pontokat, kitérve a főbb hibalehetőségekre.

# Térképek letöltése

A térképek letölthetőek innen: http://download.geofabrik.de/
Bounding box számításra szükség lesz ha csak egy saját területet akarsz megjeleníteni, egyébként nem kell: https://boundingbox.klokantech.com/
Alul a CSV formátumot érdemes választani.
Egy másik hasonló oldal: https://tools.geofabrik.de/calc/#type=geofabrik_standard&bbox=15.843,45.6407,23.0665,48.7777&tab=1&proj=EPSG:4326&places=2
Én ezt nem csináltam meg, a teljes Magyarország térképet használtam.
Ha erre szükség van, akkor használni kell az osmium parancsot is.

>     sudo apt-get install osmium-tool
>     osmium extract --bbox=6.1173598760,48.9662745077,8.5084754437,50.9404435711 --set-bounds --strategy=smart europe-latest.osm.pbf  --output rlp.osm.pbf

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

Az eredeti leírás coastline-okat is használ, én ezt nem használtam mivel csak Magyarország térképpel próbáltam. Teljes Európa térkép generálásához kellenek a coastlineok az eredeti leírás alapján.
Ha erre szükség van, akkor a tilemaker használata előtt le kell tölteni a coastlineokat a tilemaker könyvtárban állva:

>     wget https://osmdata.openstreetmap.de/download/water-polygons-split-4326.zip
>     unzip water-polygons-split-4326.zip


Használat előtt a tömörítést ki kell kapcsolni az adott konfigban, pl. tilemaker/resources/config-openmaptiles.json
Meg kell keresni a következő sort:
>     compress:"gzip"
és megváltoztatni erre:
>     compress:"none"
Ha ez nem történik meg, akkor gzip fileok lesznek a kimeneti könyvtárban, ami kliens oldalon pronlémát okoz, ismeretlen hibával elszáll. Kliens oldalon ki kellene tömöríteni a kapott fielokat, de erre nem tudtam rájönni, hogy hogyan lehet rávenni.


Használat:

>     ./tilemaker --input $HOME/tile/hungary-latest.osm.pbf --output $HOME/tile/tilemaker/hungary-nozip --process $HOME/tile/tilemaker/resources/process-openmaptiles.lua --config $HOME/tile/tilemaker/resources/config-openmaptiles.json &> $HOME/tile/tilemaker-hungary-nozip.log &

Az eredmény a $HOME/tile/tilemaker/hungary-nozip könyvtárban lesz. Ha az outputnak .mbtiles kiterjesztést adunk, akkor könyvtárstruktúra helyett SQLite adatbázist generál egy fileba. Ezt nem próbáltam, a könyvtártsruktúra levileg gyrosabb.
Itt fontos, hogy meg kell nyitni a metadata.json-t ($HOME/tile/tilemaker/hungary-nozip/metadata.json-t), és átírni az URL-t a "tiles" elem alatt. A fileok statikus fileokként lesznek kiszolgálva, olyan URL-t kell ideírni ahol elérhetőek lesznek.

>     "tiles":["http://osm.example.com/hungary-nozip/{z}/{x}/{y}.pbf"]

# Szerver

A fileok kiszolgálásához kell egy web szerver, mi most Apache-ot használunk.

## Apache install, konfig

>     sudo ufw allow 80,443,8080/tcp
>     sudo systemctl restart ufw
>     sudo apt install apache2 libapache2-mod-php
>     sudo systemctl start apache2
>     sudo systemctl enable apache2
>     sudo a2enmod proxy proxy_http headers proxy_wstunnel
>     sudo nano /etc/apache2/sites-available/openmaptiles.conf

TODO link apache/openmaptiles.conf

>     sudo a2ensite openmaptiles.conf
>     sudo apt install certbot
>     sudo apt install python3-certbot-apache
>     sudo certbot --apache --agree-tos --redirect --hsts --staple-ocsp --uir --email you@example.com -d osm.example.com
>     sudo systemctl restart apache2

## Weboldal

Létre kell hozni egy könyvtárat ahol a html és a térkép fileok lesznek, ide kell másolni a tilemakerrel létrehozott könyvtárat és a fontokat is érdemes ide betenni:

>     sudo mkdir /var/www/tileserver
>     # cp helyett mv-vel is át lehet mozgatni akkor nem foglal dupla helyet, vagy át is lehet linkelni
>     sudo cp -R $HOME/tile/tilemaker/hungary-nozip /var/www/tileserver
>     # mv alternatíva
>     # sudo mv $HOME/tile/tilemaker/hungary-nozip /var/www/tileserver
>     sudo cd /var/www/tileserver
>     sudo git clone https://github.com/klokantech/klokantech-gl-fonts fonts
>     sudo ln -sf 'KlokanTech Noto Sans Bold' fonts/Bold
>     sudo ln -sf 'KlokanTech Noto Sans Regular' fonts/Regular
>     # a json file neve meg kell egyezzen a könyvtár nevével az astrid-gunther leírás szerint (nem próbáltam más névvel)
>     sudo nano hungary-nozip.json
TODO link www/hungary-nozip.json
>     sudo nano index.html
TODO link www/index.html
>     sudo chown -R www-data /var/www/tileserver

Ami fontos a hungary-nozip.json-ben:
+ A metadata.json elérési útja relatív az index.html-hez
>     "sources": {
>       "openmaptiles": {
>         "type": "vector",
>           "url": "hungary-nozip/metadata.json"
>       }
>     },

+ Az index.html-ben a script részben a style, center és zoom argumentumokat kell jól beállítani
++ A style-ban állítható a json file elérési útja
++ A center a térkép közepe, ennek meghatározásához jól jön a leírás elején található bounding box számoló oldal
++ A zoom a kívánt zoom szint
>     <script>
>     	var map = new maplibregl.Map({
>     		container: 'map',
>     		style: 'hungary-nozip.json',
>     		center: [18, 46],
>     		zoom: 4
>     	});
>     	map.addControl(new maplibregl.NavigationControl());
>     </script>

