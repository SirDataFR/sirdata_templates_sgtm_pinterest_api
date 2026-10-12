# Optimizing Pinterest Conversions API in GTM Server-Side with Enhanced Consent and Data Handling

You can use this tag in the Google Tag Manager Server-Side container to configure the Pinterest Conversions API and manage cookies, data transmission and consent directly on the server with built-in controls.

The tag operates based on GA4 requests and supports custom settings and parameters.

The Pinterest Conversions API serves the same purpose as the Pinterest Tag, tracking user interactions on your website. However, while the Tag processes requests in the user's browser, the Conversions API operates through a cloud server, enabling various optimizations such as bypassing ad blockers and enriching events with hashed customer data.

This tag works with any container hosting service (Google Cloud, Stape, Sirdata, Taggrs), and is optimized for Sirdata's data enrichment layer, which is compatible with all hosting platforms. This additional layer of data enables several enhancements, including:
- Utilizing TCF consent signals in addition to Consent Mode signals
- Supporting a cookieless user ID when consent is not provided
- Handling consent under GDPR where applicable, and adjusting behavior where GDPR does not apply.

👉 For [Sirdata](https://sgtm.sirdata.io/login) clients, **many additional automatic optimizations are available**, such as integrated third-party ID management, enhanced consent handling, and advanced data enrichment. These features significantly improve Pinterest's Event Quality Score, boosting campaign efficiency and tracking precision.

## How it works

1. **Consent gate.** The tag fires only when personalized-ads consent is granted, detected from (in order) the Sirdata `Gtm-Helper-Consent-Personalized-Ads` header (TCF), the Consent Mode `gcs` signal (`ad_storage`), or the `Gtm-Helper-Gdpr-Applies: false` header when no Consent Mode signal is present. Without consent nothing is sent, unless the experimental cookieless mode is enabled.
2. **Event mapping.** GA4 event names are mapped to Pinterest standard events; any other name is sent as is.
3. **Enrichment.** User data is collected from GA4 `user_data`, from Sirdata `Gtm-Helper-*` headers and from the `_gtmeec` first-party cookie (shared with the [Sirdata Meta CAPI tag](https://github.com/SirDataFR/sirdata_templates_sgtm_meta_capi)), then normalized and hashed exactly as specified by Pinterest.
4. **Identifiers.** The Pinterest Click ID (`epik`) is read from the landing URL, the referrer or the `_epik` cookie, which is (re)written server-side when consent is given and the API accepted the event. The external ID is the GA4 `user_id`, hashed like the Pinterest Tag does, or Sirdata's site-only cookieless ID.
5. **Deduplication.** The GA4 `event_id` is used as the Pinterest `event_id`, so events sent by both the Pinterest Tag and this tag are deduplicated when the Tag sends the same `event_id` (use the same variable in the Pinterest Tag and in the `event_id` parameter of the GA4 event tag firing on the same trigger).

## GA4 → Pinterest event mapping

| GA4 event | Pinterest event |
| :--- | :--- |
| `page_view`, `view_item` | `page_visit` |
| `view_item_list` | `view_category` |
| `search`, `view_search_results` | `search` |
| `add_to_cart` | `add_to_cart` |
| `add_to_wishlist` | `add_to_wishlist` |
| `begin_checkout` | `initiate_checkout` |
| `add_payment_info` | `add_payment_info` |
| `purchase` | `checkout` |
| `sign_up` | `signup` |
| `generate_lead` | `lead` |
| `contact`, `customize_product`, `find_location`, `schedule`, `start_trial`, `submit_application`, `subscribe` | same name |
| anything else | same name |

`value` is sent as a string with 2 decimals at most; when the GA4 event has no value, it is computed from the items (price x quantity).

## User data sources

| Pinterest field | Sources (in priority order) | Processing |
| :--- | :--- | :--- |
| `em` | `user_data.email`, `user_data.email_address`, `email`, `user_data.sha256_email_address`, `Gtm-Helper-User-Hashed-Email`, `_gtmeec` | trim + lowercase + SHA-256 |
| `ph` | `user_data.phone_number`, `phone_number`, `user_data.sha256_phone_number`, `Gtm-Helper-User-Hashed-Phone`, `_gtmeec` | digits only, leading zeros removed, 8-15 digits, SHA-256 |
| `fn` / `ln` | `user_data.address.first_name` / `last_name` (or `sha256_*`) | trim + lowercase + SHA-256 |
| `ct` | `user_data.address.city`, then `Gtm-Helper-User-City` | lowercase, whitespace and ASCII punctuation removed, SHA-256 |
| `st`, `zp`, `country` | `user_data.address.region`, `postal_code`, `country` (country falls back on `Gtm-Helper-User-Country`) | lowercase (zip code without spaces or dashes, 2-letter country code), SHA-256 |
| `ge`, `db` | `user_data.gender`, `user_data.date_of_birth` | `f`/`m`/`n`, `YYYYMMDD`, SHA-256 |
| `external_id` | `external_id`, `user_id`, `Gtm-Helper-Cookieless-Id-Domain-Specific`, `_gtmeec` | trim + lowercase + SHA-256 (as the Pinterest Tag does) |
| `click_id` | `epik` parameter of the landing URL or referrer, `_epik` cookie, `click_id` | raw |
| `client_ip_address` | `Gtm-Helper-User-Ip`, `ip_override` | raw |
| `client_user_agent` | `Gtm-Helper-Device-User-Agent`, `user_agent`, `User-Agent` | raw |

Already hashed values (64 hex characters) are never re-hashed. Values equal to `redacted_for_privacy` are ignored.

## Cookies

| Cookie | Purpose | Lifetime | HttpOnly |
| :--- | :--- | :--- | :--- |
| `_epik` | Pinterest Click ID, compatible with the Pinterest Tag | 1 year | no |
| `_gtmeec` | Enrichment record (hashed email, hashed phone, external ID) shared with the Sirdata Meta CAPI tag | 1 year | yes |

Cookies are written only when consent is given and the **Generate cookies** option is enabled (`_epik` once Pinterest returned a 2xx response).

## Installation

1. In your GTM Server container, go to **Templates** → **Tag Templates** → **New**, open the three-dot menu, choose **Import** and select `template.tpl`.
2. Create a new tag using the **GDPR Ready Pinterest CAPI by Sirdata** template.
3. Fill in the **API Access Token** and the **Pinterest Ad Account ID** (from the Conversions section of Pinterest Ads Manager).
4. Trigger the tag on the GA4 events you want to send (e.g. all GA4 events with **Inherit from GA4**). If another Pinterest Conversions API tag is already in place, pause it to avoid sending events twice.

Use **Test mode** to test the integration without recording conversions.

## Configuration

| Parameter | Description |
| :--- | :--- |
| **Event Name Setup Method** | Inherit from GA4 (default), a fixed Pinterest standard event, or a custom event name. |
| **API Access Token** | Mandatory. |
| **Pinterest Ad Account ID** | Mandatory. Also called Advertiser ID; this is not the Pinterest Tag ID. |
| **Test mode** | Adds `test=true` to the request; events are validated but not recorded. |
| **Generate cookies** | Writes `_epik` and `_gtmeec` when consent is given. |
| **Send hashed user data** | Enables email, phone, name, address and `_gtmeec` enrichment. |
| **Send key identifiers** | Enables IP, user agent, external ID and coarse location. |
| **Send without consent (experimental)** | Without consent, sends events with `opt_out: true` and only the Sirdata site-only cookieless ID (hashed), the click ID found in the URL, IP, user agent (required by Pinterest) and coarse location. No cookie is read or written, no user data is sent. |
| **Item ID key** | GA4 item property used as `content_ids` and `contents[].id` (default `item_id`). |
| **Server Event Data Override** | Force `event_id`, `event_source_url`, `event_time`, `action_source`, `opt_out` or `language`. |
| **User Data** | Add or override any `user_data` field; values may be single values or JSON arrays (3 values max per field), hashing is applied automatically. |
| **Custom Data** | Add or override any `custom_data` field (`value`, `currency`, `content_ids`, `contents`, `num_items`, `order_id`, `search_string`, `opt_out_type`, etc.). |

## Useful resources

- [Pinterest Conversions API](https://developers.pinterest.com/docs/api-features/conversion-overview/)
- [Pinterest Conversions API events reference](https://developers.pinterest.com/docs/api/v5/events-create)
- [Sirdata GTM Helper layer](https://server-side.docs.sirdata.net/sirdata-server-side/english-1/installation/data-processing/gtm-helper-layer)
- [GDPR Ready Meta/Facebook CAPI by Sirdata](https://github.com/SirDataFR/sirdata_templates_sgtm_meta_capi)

## License

Apache 2.0 — Copyright 2026 Sirdata (sirdata.com).
