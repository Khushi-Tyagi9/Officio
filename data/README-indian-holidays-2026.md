# indian-holidays-2026.csv: source, status and caveats

Import file for the `contoso_holiday` table (columns: `Name`, `Holiday Date`, `Holiday Type`, `Working Day?`). Checked on 22 September 2026.

## Source

Government of India, Ministry of Personnel, Public Grievances and Pensions (Department of Personnel and Training), **Office Memorandum F.No.12/2/2023-JCA dated 3 July 2025**, "Holidays to be observed in Central Government Offices during the year 2026":
<https://dopt.gov.in/sites/default/files/Holidays%20to%20be%20observed%20in%20Central%20Government%20Offices%20during%20the%20year%202026.pdf>

The PDF is a scanned image (six pages). It was read visually, not from extracted text: the memorandum (paras 1-11), Annexure-I (gazetted holidays, Delhi/New Delhi offices) and Annexure-II (restricted holidays). Every date and weekday below was compared with Annexure-I.

## What is in the CSV: the 14 holidays that are compulsory everywhere

Para 2 of the memorandum lists 14 holidays that offices *outside* Delhi must observe compulsorily. Those are the all-India holidays, and they are the rows in the CSV. All 14 carry the type "National Public Holiday" and `Working Day?` = No.

| Holiday | Date | Day | Status |
|---|---|---|---|
| Republic Day | 2026-01-26 | Mon | Fixed in the memorandum |
| Id-ul-Fitr | 2026-03-21 | Sat | **Provisional (moon sighting).** Press reports say Eid was observed in India on 21 March; no revision found. |
| Mahavir Jayanti | 2026-03-31 | Tue | Fixed |
| Good Friday | 2026-04-03 | Fri | Fixed |
| Buddha Purnima | 2026-05-01 | Fri | Fixed |
| Id-ul-Zuha (Bakrid) | **2026-05-28** | Thu | **Provisional (moon sighting) and revised.** Annexure-I says 27 May (Wed). A later DoPT change notice, dated 22 May 2026 and reported by several outlets, moved the holiday for Delhi/New Delhi offices to 28 May. See the note below. |
| Muharram | 2026-06-26 | Fri | **Provisional (moon sighting).** No revision notice found; not independently confirmed after the fact. |
| Independence Day | 2026-08-15 | Sat | Fixed |
| Milad-un-Nabi (Id-e-Milad) | 2026-08-26 | Wed | **Provisional (moon sighting).** No central revision found; reports of Andhra Pradesh and Telangana orders also use 26 August. |
| Mahatma Gandhi Jayanti | 2026-10-02 | Fri | Fixed |
| Dussehra | 2026-10-20 | Tue | Fixed |
| Diwali (Deepavali) | 2026-11-08 | Sun | Fixed in the memorandum. Para 6 lets some states observe "Naraka Chaturdasi" instead. |
| Guru Nanak Jayanti | 2026-11-24 | Tue | Fixed |
| Christmas Day | 2026-12-25 | Fri | Fixed |

### Moon-sighting holidays

Para 5.1 says any change to the dates of **Id-ul-Fitr, Id-ul-Zuha, Muharram and Id-e-Milad**, "depending upon sighting of the Moon", is declared by the Ministry (for Delhi offices). Para 5.2 lets the state coordination committees change them for offices outside Delhi, and para 5.3 allows short-notice announcements through the press and broadcast media. So those four dates in the file are the memorandum's dates, not guaranteed final dates. Offices outside Delhi may observe them a day apart.

**Bakrid, what was and was not checked.** The 28 May date comes from a DoPT change notice dated 22 May 2026 (PIB release "Change in date of holiday on account of Id-ul-Zuha (Bakrid)", PRID 2264022) and from press coverage such as [People Matters](https://www.peoplematters.in/news/business/bakrid-2026-holiday-moved-from-may-27-to-may-28-says-centre-49897). The search excerpt of the notice reads that Delhi/New Delhi offices "shall remain closed on 28th May, 2026 ... (in place of 27th May, 2026)". I could **not** open the notice itself: the PIB page refuses automated requests, and the DoPT-hosted change notice I downloaded is a scanned image I could not decode. Treat 28 May as reported but not read from the primary document, and confirm it before relying on it for payroll.

## Left out on purpose

- **Holi (4 Mar), Ram Navami (26 Mar), Janmashtami (4 Sep).** They appear in Annexure-I only because they are Delhi's three picks from the memorandum's list of 12 *optional* holidays (para 3.1). Offices outside Delhi choose their own three, so these are state-specific in practice. Add them as rows (type "Other", or "National Public Holiday" if your organisation follows Delhi) if you want the Delhi list.
- **Restricted holidays (Annexure-II, 34 items).** Employees choose two; they are not days off for everyone. Dr B.R. Ambedkar Jayanti, which one secondary website lists as gazetted, appears in neither annexure.

## Notes for use

- Three holidays fall on a weekend: Id-ul-Fitr (Saturday), Independence Day (Saturday) and Diwali (Sunday). Para 3.2 says no substitute holiday is given when a festival falls on a weekly off. The leave app should therefore not deduct a weekend holiday from a leave request (see `docs/bug-fixes.md`, BUG-001).
- Scope: this is the **Central Government** list. It is not a substitute for your own employer's policy, and it does not cover banks (para 10 refers those to the Department of Financial Services).
- The memorandum is dated July 2025. Later notices for the moon-sighting holidays were searched for but not exhaustively.
- Import: use the Import Data wizard; dates are ISO (`yyyy-MM-dd`). If the wizard rejects them, reformat to your regional date format. The importing user becomes the row owner.
