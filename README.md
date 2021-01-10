# protobeats

Protobeats is a Protobuf-based musical messaging format. It
is designed to store data in a format very similar to raw 
(over-the-wire, as opposed to file-based) MIDI, while adding 
the basics of DAW software features and an "MVP" subset of music 
engraving app features (like Sibelius, MuseScore, etc.). All while
being [MIT-licensed](LICENSE.md), easy for others (you, reading this)
to read and extend and build things with, and 

## Usage
Protobeats data can be included in your service Protos, written to any USB, Bluetooth
or other serial byte buffer, transformed to/from JSON, XML, or anything else, and generally 
be used in all the ways you'd use Protobuf models.
### Dart/Flutter: File I/O
If you have a `beatscratch` file (you can find this folder on macOS or iOS by running a MIDI Export and going up a folder in Finder/Files when it completes), this Protofile is all you need to read it.

```dart
static Future<Score> loadScore(File file) async =>
      Score.fromBuffer(file.readAsBytesSync());

static saveScore(File scoreFile, Score score) async =>
    scoreFile.writeAsBytes(score.deepCopy().writeToBuffer());
```

### Dart/Flutter: Score URLs
In addition to `music.proto` you'll need a Base58 encoder/decoder and a BZip2 tool. A BeatScratch in-app URL can be decoded in one of two ways:

* **Short URL** (*beatscratch.io/app/#/s/`<id>`*) these are stored with proprietary code on https://paste.ee in BeatScratch.
    * You can get the Long URL by visiting paste.ee/p/`<id>` (i.e. beatscratch.io/app/#/s/TgdNH becomes paste.ee/p/TgdNH 
* **Long URL** (*beatscratch.io/app/#/score/`<encoded_data>`*) these are stored with proprietary code on https://paste.ee in BeatScratch.
    * `<encoded_data>` *may or may not* be BZip2 compressed with [archive](https://pub.dev/packages/archive) (depending on what is longer). The binary data is then Base58-encoded with [base_x](https://pub.dev/packages/base_x). To read a long-form URL, you'll need to Base58-decode, then try reading a Score, finally falling back to decompressing and reading the Score.

```dart
import 'dart:convert';

import 'package:archive/archive.dart';
import 'package:base_x/base_x.dart';
import 'package:http/http.dart' as http;

import '../generated/protos/music.pb.dart';

var _base58 = BaseXCodec(
    '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ');
final urlPrefix = kIsDebug && false
    ? "http://localhost:8000/app-staging/"
    : "https://beatscratch.io/app/";

extension URLConversions on Score {
  String convertToUrl() {
    final dataString = toUrlHashValue();
    final urlString = "$urlPrefix#/score/$dataString";
    return urlString;
  }

  Future<String> convertToShortUrl() async {
    try {
      http.Response response = await http.post(
        'https://api.paste.ee/v1/pastes',
        headers: <String, String>{
          "X-Auth-Token": "<my-pastee-token>",
          'Content-Type': 'application/json; charset=UTF-8',
        },
        body: jsonEncode(<String, dynamic>{
          'expiration': 31536000,
          'sections': [
            {'contents': convertToUrl()}
          ],
        }),
      );
      String pasteeeShortcode = jsonDecode(response.body)['id'];
      final urlString = "$urlPrefix#/s/$pasteeeShortcode";
      return urlString;
    } catch (any) {
      return null;
    }
  }

  String toUrlHashValue() {
    final data = writeToBuffer();
    final bz2Data = BZip2Encoder().encode(data);
    final dataToConvert = (data.length <= bz2Data.length) ? data : bz2Data;
    final dataString = _base58.encode(dataToConvert);
    return dataString;
  }
}

Score scoreFromUrlHashValue(String urlString) {
  final dataBytes = _base58.decode(urlString);
  Score score;
  try {
    score = Score.fromBuffer(dataBytes);
  } catch (any) {
    try {
      score = Score.fromBuffer(BZip2Decoder().decodeBytes(dataBytes));
    } catch (any) {}
  }
  return score;
}

```

## Contribute, Compete, Cooperate
Please, build an app better than BeatScratch with this format! None of my UI
ideas are patented, but you gotta write your own code. Fuck software patents.
Or build a music storage service to sell to people! I'll happily integrate it 
for you. Note that you'll need to figure out licensing in that case. Please
do contribute back changes to the `music.proto` if possible though. You don't
have to, of course! But being simple and open is kind of why you'd use it, no?

### Future features
Observant developers/musicians may note that Protobeats has SoundFont support
that is not yet implemented in BeatScratch. Eventually synthesizer-related features like
compression, delay, reverb should be supported. Please, fork and PR how you'd prefer
to do this - it's just one Protofile after all!

If there is enough interest, I will add a `protobeats_plugin` repo that adds a *lot* 
of cool language support for your Dart/Flutter apps.