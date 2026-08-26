# Attribution

This project (sparks-crosswalk-data) aggregates identifiers from multiple open and community-driven public sources.

## Sources

| Source | Used for | Terms |
| --- | --- | --- |
| [Wikidata](https://www.wikidata.org) | `wikidata_uri` / `wikidata_id` on every dataset | [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/) |
| [IPTC Media Topics](https://cv.iptc.org/newscodes/mediatopic/) | `iptc_media_topic_uri` on sports | [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) |
| ESPN core API | `espn_api_uri` on sports and leagues, `espn_id` on venues | Public endpoints, no published license |
| [Reuters sports slugs](https://liaison.reuters.com/tools/sports-slugs) | `reuters_sport_slug`, `reuters_league_slug` | Published reference list |
| MLB StatsAPI | `mlb_id` on venues | Public endpoints; see MLB's [copyright notice](http://gdx.mlb.com/components/copyright.txt) |

Geographic coordinates, founding years, and venue naming history are drawn primarily from Wikidata and
cross-checked against the operators' own published material.

## General notes

- Wherever possible, contributions to this project aim to respect original licenses and terms of use.
- This project does not redistribute raw source data in bulk, only normalized cross-reference mappings.
- Identifiers themselves are facts, not creative works, and are recorded here to make sources
  interoperable rather than to substitute for any of them.
- If you maintain one of the sources above and want its attribution corrected or its identifiers
  removed, please open an issue.
