# Code Change Document (CCD)

## Z_SCM_TRIGGER_INDENT — Remove SAP date/time gate before indent API + Option B

| Field | Value |
|--------|--------|
| **Program** | `Z_SCM_TRIGGER_INDENT` |
| **Primary include (logic)** | `Z_SCM_TRIGGER_INDENT_CLASS` |
| **Global data include** | `Z_SCM_TRIGGER_INDENT_TOP` |
| **Document version** | 2.1 |
| **Document date** | 06.05.2026 |
| **Target stack** | SAP ECC 6.0 / NetWeaver 7.31 (`abap_731`) |
| **Related rules** | `ABAP Rules - 02-04-2026/` (see §2) |
| **Related FS** | `Functional Specification - Z_SCM_TRIGGER_INDENT no API date-time gate Option B.md` (v2.0) |
| **Deliverable scope** | **This CCD and the related FS are the specification deliverables.** ABAP changes are applied in **SAP** (includes above); no other workspace file is mandated by this CCD. |

**Revision history**

| Version | Summary |
|---------|---------|
| 1.1 | Superseded: date-only API gate `GE sy-datum`. |
| 1.2 | Superseded: Option B + date-only gate + reference to workspace export. |
| 2.0 | **Full removal** of SAP **date/time** gate before **`Z_SAP_TO_REST_API_NEW`**; **Option B**; assumptions on ship-flag/`VTTK` data; implementer **SE80** dependency check; **FS v2.0** only. |
| 2.1 | CCD file renamed from `Code Change Document - Z_SCM_TRIGGER_INDENT SVPT date-only gate.md` to `Code Change Document - Z_SCM_TRIGGER_INDENT no API date-time gate Option B.md` (align with FS); technical content unchanged from v2.0. |

---

## 1. Business requirement

**Current behavior:** Before **`Z_SAP_TO_REST_API_NEW`**, the program applies a **date + time** check using **`gv_finaldate`**, **`gv_finaltime`**, packed **`gv_timestamp`** vs **`sy-datum`** + **`sy-uzeit`**, which can delay or block the API on the same calendar day until clock time passes, or block past scheduled dates.

**Desired behavior (Option A — full removal):** **No** SAP-side **date** or **time** condition shall determine whether **`Z_SAP_TO_REST_API_NEW`** is invoked. When execution reaches the call (subject to **unchanged** upstream logic such as token success and duplicate-success checks), the function module **runs**.

**Option B:** Do **not** set **`gv_svtp_date_error`** (text-021) when **`gv_date_t LT sy-datum`**, so **`zlog_int_db`**, **`gv_error_msg`** / **`UPDATE err_desc`**, and ALV status are **not** overridden by a SAP “past placement date” error while the platform owns validation of payload dates.

**Payload (3.4):** Removing the SAP API gate **does not** remove past (or any) dates from the outbound payload; **`gw_api_input-vehicle_placement_date_time`** (and related construction) **remains**; the **SCM platform** validates.

**Data assumptions (3.1):** Business confirms ship-flag / **`VTTK`** sources will **not** be blank or `00000000` for valid in-scope shipments in normal operation. This CCD does **not** add a new SAP guard for missing schedule.

**Field usage (3.2):** It is **unknown** whether **`gv_finaldate`**, **`gv_finaltime`**, or the **`gt_zlog_ship_flag_tmp`** read exist **only** for the API gate. **Implementer:** search **SE80** across all includes; **remove dead code** only after confirmation. If still used elsewhere, **remove only** the **`IF`** nesting around **`CALL FUNCTION`**, not necessarily the population block.

---

## 2. Compliance with ABAP Rules (`ABAP Rules - 02-04-2026/`)

| Rule file | Application to this change |
|-----------|----------------------------|
| `00-main.mdc` | NetWeaver **7.31** compatible syntax only. |
| `12-documentation.mdc` | Program / include **change history**; short inline comment: **no SAP date/time gate before indent API** + **Option B**. |
| `14-transport.mdc` | One transport; clear description; related includes together. |
| `08-testing.mdc` | Tests per FS AT-xxx; **SCI** / **SLIN** clean. |
| `11-interfaces-api.mdc` | No FM interface change; call placement / wrapping changes only. |

---

## 3. Objects in scope (transport)

| Object type | Object name | Remark |
|-------------|--------------|--------|
| Report | `Z_SCM_TRIGGER_INDENT` | Main program |
| Include | `Z_SCM_TRIGGER_INDENT_CLASS` | **Mandatory:** remove API date/time **`IF`**(s); **`CALL FUNCTION 'Z_SAP_TO_REST_API_NEW'`** unguarded by date/time per FS; **Option B** |
| Include | `Z_SCM_TRIGGER_INDENT_TOP` | **Conditional:** remove **`gv_timestamp`** / **`gv_sytm_timestamp`** **only if** unused after gate removal (verify SE80) |
| Other includes | As found by search | Any duplicate gate logic |

**Suggested transport description (example):**

`<TICKET-ID>: Z_SCM_TRIGGER_INDENT — remove SAP date/time gate before indent API; Option B (no gv_svtp_date_error for gv_date_t LT sy-datum); CLASS/TOP per analysis`

---

## 4. Technical design

### 4.1 Remove — API date/time gate (`Z_SCM_TRIGGER_INDENT_CLASS`)

**Historical pattern (to remove):**

```abap
IF ( ( gv_finaldate NE '00000000' ) AND
     ( gv_finaltime NE '' ) ).
  gv_timestamp = gv_finaldate && gv_finaltime.
  gv_sytm_timestamp = sy-datum && sy-uzeit.
  IF gv_timestamp GE gv_sytm_timestamp.
    CALL FUNCTION 'Z_SAP_TO_REST_API_NEW' ...
  ENDIF.
ENDIF.
```

**Any intermediate variant** (e.g. date-only `IF gv_finaldate GE sy-datum`) is **also removed** under FS v2.0 — there is **no** SAP date/time eligibility `IF` for this API.

**Target pattern:**

```abap
CALL FUNCTION 'Z_SAP_TO_REST_API_NEW'
  EXPORTING
    im_data                    = gw_api_input
    im_url                     = gv_platform_url
    im_http_method             = 'POST'
    it_header                  = gt_token
    im_data_new                = gw_api_input
  IMPORTING
    ex_reponse                 = gw_reponse
    ex_response_status         = gw_reponse_status
  EXCEPTIONS
    ...
```

Place the **`CALL FUNCTION`** at the same logical position as today (after token / header preparation succeeds), **without** wrapping **`IF`** on **`gv_finaldate`**, **`sy-datum`**, **`sy-uzeit`**, or packed timestamps.

### 4.2 Option B — `gv_svtp_date_error` (`Z_SCM_TRIGGER_INDENT_CLASS`)

**Remove:**

```abap
IF gv_date_t < sy-datum.
  gv_svtp_date_error = text-021.
ENDIF.
```

**Replace** with a brief comment documenting **Option B** (no assignment for this case). Retain **`DATA gv_svtp_date_error`** and loop **`CLEAR`** unless a later FS retires the field.

### 4.3 Optional cleanup (after SE80 verification)

| Item | Action |
|------|--------|
| `gv_timestamp`, `gv_sytm_timestamp` | Remove from **TOP** and all **`CLEAR`** / assignments **if** only used for removed gate |
| `gv_finaldate` / `gv_finaltime` / `READ gt_zlog_ship_flag_tmp` block | Remove **only if** proven unused after search; else keep for payload or other logic |

---

## 5. Behavioral impact

| Scenario | Before | After (v2.0) |
|----------|--------|----------------|
| Same calendar day, time before scheduled time | API could be **blocked** | API **not** blocked by SAP time |
| **`gv_finaldate` LT `sy-datum`** | API **blocked** by gate | API **not** blocked by SAP date gate |
| **`gv_date_t` LT `sy-datum`** | **`gv_svtp_date_error`** = text-021 | **Option B:** error **not** set; messaging from API / other paths |
| Platform rejects payload | May never have been called from SAP | SAP may call; **platform** returns error |

---

## 6. Documentation updates (`12-documentation.mdc`)

- [ ] Program / include **change history** (date, user, ticket).
- [ ] Inline comment at **`CALL FUNCTION`**: no SAP date/time gate per FS v2.0.
- [ ] Inline comment at **Option B** removal site.

---

## 7. Test plan (`08-testing.mdc`)

Align with FS **AT-001** … **AT-007**. Minimum:

| # | Test case | Expected |
|---|------------|----------|
| 1 | Token OK, shipment eligible | **`Z_SAP_TO_REST_API_NEW`** called **without** date/time `IF` |
| 2 | Past **`gv_finaldate`** vs `sy-datum` (if populated) | API **still** called |
| 3 | **`gv_date_t` LT `sy-datum`** | **`gv_svtp_date_error`** initial; no text-021 from this rule |
| 4 | Prior indent success | Row skipped (**unchanged**) |
| 5 | Payload with past epoch | POST sent; outcome per **platform** |
| 6 | SCI + SLIN | Clean |

---

## 8. Sign-off

| Role | Name | Signature / Date |
|------|------|------------------|
| Functional owner | | |
| ABAP developer | | |
| Peer reviewer | | |

---

## 9. Appendix

- **FS:** `Functional Specification - Z_SCM_TRIGGER_INDENT no API date-time gate Option B.md` (v2.0).
- **Superseded FS** removed from workspace: `Functional Specification - Z_SCM_TRIGGER_INDENT SVPT date-only GE sy-datum.md` (replaced by v2.0 file above).
- **CCD filename:** This document was formerly `Code Change Document - Z_SCM_TRIGGER_INDENT SVPT date-only gate.md` (renamed to match FS v2.0 scope).
