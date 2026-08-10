# us-ca-tax-conformance

A vendor-neutral set of conformance cases for US sales tax rate resolution, using California as the worked example. Plain data, no dependency on any tax provider, CC0.

The point is to turn "our tax model cannot express a US local rate" from something a user discovers in production into a failing test.

## Why California

California is the hardest common case in the US and the easiest to verify, because CDTFA publishes jurisdiction rates as open data with no use restrictions. It is also not a Streamlined Sales Tax member state, so the SST boundary and rate files that cover many states do not cover it. CDTFA is the only primary source.

## What the cases prove

Ten cases, each with the assertion, why it matters, and the specific failure mode:

| Case | What breaks |
| --- | --- |
| `enclave-piedmont-in-oakland` | Piedmont (10.25%) is a city entirely surrounded by Oakland (10.75%). Nearest-city or ZIP-centroid fallback silently returns the wrong rate. |
| `three-rates-within-two-miles` | Berkeley 10.25%, Emeryville 10.50%, Oakland 10.75%, all contiguous. |
| `adjacent-culver-city-vs-los-angeles` | 100 bp turns on a municipal line inside one postal metro. |
| `adjacent-santa-monica-vs-los-angeles` | Second independent 100 bp instance on another border of the same city. |
| `enclave-west-hollywood-in-los-angeles` | Shares ZIP territory with Los Angeles at a different rate, so ZIP-level lookup has no correct answer. |
| `unincorporated-vs-incorporated-alameda` | Postal city name is not a tax jurisdiction. |
| `intra-county-spread-los-angeles` | 89 jurisdictions, 150 bp spread. One-rate-per-county models cannot represent it. |
| `half-basis-point-rates` | **61 California jurisdictions** have rates that are not whole basis points. Integer `rate_bp` cannot store them. |
| `float-rate-multiplication-artifact` | `0.085 * 10000` is `850.0000000000001` in float64. Truncation gives 849. |
| `two-rate-quarters-live-simultaneously` | The source itself publishes two effective quarters at once. Rate lookup is a function of location *and* date. |

Two of these are worth calling out because they are usually argued from anecdote and here they are counted from the state's own data.

**Integer basis points are insufficient, and not marginally.** 61 jurisdictions across eight counties carry a half-basis-point component, including the City and County of San Francisco at 8.625% and most of San Mateo County. 862.5 bp is not storable as an integer; 862 understates and 863 overstates. Parts per million works (8.625% is exactly 86,250 ppm) but pick more headroom than today's worst case, because granularity is set by thousands of local jurisdictions that coordinate with nobody.

**A single combined rate is the wrong thing to store anyway.** A US return is filed per jurisdiction, with state and local portions reported separately. A field holding only the combined figure has already discarded what a return needs. Widening the precision fixes the error message and leaves that gap untouched. Consider an ordered component list with the combined rate derived rather than stored, because retrofitting after documents have been issued is far worse than doing it now.

## Scope and honest limitations

- **Combined rates only.** CDTFA's public layer publishes the combined rate per jurisdiction. It does not break out state, county and district components. The cases here assert combined rates. If you need component splits, CDTFA-105 (District Sales and Use Tax Rates) is the authority, and a contribution adding those would be welcome.
- **Jurisdiction level, not rooftop.** Cases assert which jurisdictions exist and what they charge. They do not assert that a specific street address resolves to a specific jurisdiction, because that requires a point-in-polygon geocode that cannot be verified from attribute data alone. Asserting it without doing the geometry would make this file confidently wrong, which is worse than narrow. See below.
- **One state.** The failure modes generalise; the data does not.
- **A vintage, not a feed.** Rates change quarterly. Every case carries the effective date. Re-derive rather than trusting a stale copy.

## Provenance

| | |
| --- | --- |
| Authority | California Department of Tax and Fee Administration |
| Dataset | `CDTFA_SalesandUseTaxRates_Public`, layer 1 |
| Landing page | https://data.ca.gov/dataset/cdtfa-salesandusetaxrates-public |
| Rights | "No restrictions on public use" |
| Rates effective | 2026-07-01 |
| Annexations filed through | 2026-05-01 |
| Source last published | 2026-06-15 |
| Retrieved | 2026-08-10 |

## Regenerating

Every figure came from the published feature service and can be re-derived. Base URL:

```
https://services6.arcgis.com/snwvZ3EmaoXJiugR/arcgis/rest/services/California_Sales_and_Use_Tax_Rates/FeatureServer/1
```

All jurisdictions and rates in one county:

```
/query?where=County_name='ALAMEDA'&outFields=JURIS_NAME,RATE&returnGeometry=false&f=json
```

Every jurisdiction whose rate is not a whole basis point (note: this returns three false positives at exactly 8.5%, caused by the float artifact in `float-rate-multiplication-artifact`, so 64 rows means 61 genuine):

```
/query?where=RATE*10000<>FLOOR(RATE*10000)&outFields=JURIS_NAME,County_name,RATE&returnGeometry=false&f=json
```

Per the source's own instruction, filter on `DateStamp` when two quarters are live:

```
DateStamp = (SELECT MAX(DateStamp) FROM CDTFA_SalesandUseTaxRates WHERE DateStamp < CURRENT_DATE)
```

## Extending to rooftop

The deliberate gap. To assert address-level expectations, fetch the layer geometry (GeoJSON or shapefile from the landing page), geocode the addresses, and run point-in-polygon against the jurisdiction polygons. Publish the geocoder and its vintage alongside the results, because a rooftop assertion is only as good as the geocode behind it and two geocoders disagree at exactly the boundaries these cases are about.

If you add that, keep it in a separate file so the jurisdiction-level assertions stay independently verifiable from attribute data.

## Using it

The cases are data, not a test harness, so they drop into whatever runner you already have. The useful pattern is a table-driven test per case asserting your resolver returns the expected jurisdiction and rate, plus a negative assertion for the enclave cases that it does *not* return the surrounding jurisdiction's rate. The enclave and half-basis-point cases are the ones that actually catch bugs.

## Contributing

Additional states welcome, especially ones with a comparable open dataset. Component-level breakdowns from CDTFA-105 welcome. Corrections most welcome: if a figure here disagrees with CDTFA, CDTFA is right and this file is stale.

## Who wrote this and why

Assembled by Eric Lakich. Disclosure: I work at TaxCloud, which runs Ziptax (zip.tax), a US sales tax rates API. I have an obvious commercial interest in people concluding that US rate resolution is hard.

So this file contains no reference to our API, no benchmark that favours us, and nothing you need an account to verify. Every number traces to a state government dataset you can query yourself, and the regeneration commands are above precisely so you do not have to take my word for any of it. If it ever reads like a sales argument, that is a bug worth filing.

Prior art and the reason this exists in this shape: [`PatrickPi1312/eu-vat-conformance`](https://github.com/PatrickPi1312/eu-vat-conformance) does the same job for EU VAT, 19 cases, CC0. This is the US half of that idea. His `b2c-digital-threshold-unknown-at-de` case, where the expected output is `threshold_dependent` because no correct number exists, is the sharpest thing in either file: a provider contract that can only return a number cannot express its own limits.

## License

CC0 1.0. Public domain dedication. Copy it, vendor it, strip the attribution, no need to ask.

Underlying rate data is published by CDTFA under "no restrictions on public use."
