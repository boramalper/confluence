confluence
==========

Confluence is a torrent client as a HTTP service. This allows for easy use from other processes, languages, and machines, due to the ubiquity of HTTP. It makes use of [anacrolix/torrent](https://github.com/anacrolix/torrent)'s [download-on-demand](https://godoc.org/github.com/anacrolix/torrent#Torrent.NewReader) torrenting, and [custom data backend](https://godoc.org/github.com/anacrolix/torrent/storage#ClientImpl) features to store data in a cache. You can then utilize the BitTorrent network with sensible defaults as though it were just regular HTTP.

Installation
============

```
go get github.com/anacrolix/confluence
```

## Android

See https://github.com/arranlomas/Android-Confluence-Wrapper and https://github.com/arranlomas/confluence.

Usage
=====

```
$ godo github.com/anacrolix/confluence -h
Usage:
  confluence [OPTIONS...]
Options:
  -addr                    (string)          HTTP listen address (Default: localhost:8080)
  -cacheCapacity           (tagflag.Bytes)   Data cache capacity (Default: 11 GB)
  -collectCamouflageData   (bool)
  -debugOnMain             (bool)            Expose default serve mux /debug/ endpoints over http
  -dht                     (bool)            Default: true
  -disableTrackers         (bool)            Disables all trackers
  -fileDir                 (string)          File-based storage directory, overrides piece storage
  -implicitTracker         ([]string)        Trackers to be used for all torrents
  -overrideTrackers        (bool)            Only use implied trackers
  -pex                     (bool)            Default: true
  -publicIp4               (net.IP)          Public IPv4 address
  -publicIp6               (net.IP)          Public IPv6 address
  -seed                    (bool)            Seed data
  -sqliteStorage           (*string)
  -tcpPeers                (bool)            Allow TCP peers (Default: true)
  -torrentGrace            (time.Duration)   How long to wait to drop a torrent after its last request (Default: 1m0s)
  -uPnPPortForwarding      (bool)            Port forward via UPnP
  -unlimitedCache          (bool)            Don't limit cache capacity
  -utpPeers                (bool)            Allow uTP peers (Default: true)
```

Confluence will announce itself to DHT, and wait for HTTP activity. Torrents are added to the client as needed. Without an active request on a torrent, it is kicked from the client after the torrent grace period. Its data however may remain in the cache for future uses of that torrent.

Routes
======

There are several routes to interact with torrents:

-	`GET /data?ih=<infohash in hex>&path=<display path of file declared in torrent info>`. Note that this handler supports HTTP range requests for bytes. Response will block until the data is available.
-	`GET /status`. This fetches the textual status info page per anacrolix/torrent.Client.WriteStatus. Very useful for debugging.
-	`GET /info?ih=<infohash in hex>`. This returns the info bytes for the matching torrent. It's useful if the caller needs to know about the torrent, such as what files it contains. It will block until the info is available. The response is the full bencoded info dictionary per [BEP 3](http://www.bittorrent.org/beps/bep_0003.html).
-	`/events?ih=<infohash in hex>`. This is a websocket that emits frames with [confluence.Event] encoded as JSON for the torrent. The PieceChanged field for instance is set if the given piece changed [state](https://godoc.org/github.com/anacrolix/torrent#PieceState) within the torrent.
-	`GET /fileState?ih=<infohash in hex>&path=<display path of file declared in torrent info>`. Returns [file state](https://godoc.org/github.com/anacrolix/torrent#File.State) encoded as JSON.
-	`POST /metainfo?ih=<infohash in hex>`. The request body is a bencoded metainfo, as typically appears in a `.torrent` file. The trackers and info bytes are applied to the torrent matching the info hash provided in the query. No fields in the metainfo are mandatory.
-	`GET /metainfo?ih=<infohash in hex>`. returns a .torrent file containing the hash info.

Wherever a `?ih=<infohash>` query parameter is expected, it can also be substituted by a `?magnet=<magnet URI>` parameter instead.

## Example
Running the following command from a shell will launch VLC and start playing the Sintel movie stream from its public torrent.
```
vlc 'http://localhost:8080/data?magnet=magnet:?xt=urn:btih:08ada5a7a6183aae1e09d831df6748d566095a10&path=Sintel.mp4'
```
