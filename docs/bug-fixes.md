# Bug fixes to apply in Power Apps Studio

**Nothing here has been applied.** The files in `apps_src/` were unpacked from the `.msapp` binaries, so editing them changes nothing in the real app. Make these edits by hand in Power Apps Studio (Request Leave app).

**How this was checked.** I ported the app's day-counting logic to Python and compared it with a brute-force count over 20,000 random date ranges in 2026. The Power Fx formulas below have **not** been run in Studio; Studio may want small syntax adjustments, and you should re-run the test table at the end after applying them.

Both bugs are in one place: the `OnSelect` of `LeaveStartDatePicker` on `LeaveDatesScreen` (`apps_src/requestleave/Src/LeaveDatesScreen.fx.yaml`, starting line 224). `LeaveStartDatePicker.OnChange`, `LeaveEndDatePicker.OnChange` and `LeaveEndDatePicker.OnSelect` all just call `Select(LeaveStartDatePicker)` (lines 223, 309, 310), so that is the only place to edit.

---

## BUG-001: a holiday on a weekend is subtracted from the leave days

### Current code

Lines 251-254 (line 250 is only the `);` that closes the weekday `If` chain, so the faulty statements are 251-253, not 250-253):

```
251  Set(_workDaysInRequest, _numFullWeeks * 5 + _numPartialWeekdays);
252  Set(_holidaysInRequest, CountIf(HolidaysCollection, 'Holiday Date' >= LeaveStartDatePicker.SelectedDate, 'Holiday Date' <= LeaveEndDatePicker.SelectedDate));
253  Set(_requestedDays, _workDaysInRequest - _holidaysInRequest);
254  If(_requestedDays = 1 && drpDayPeriod.Visible, Select(drpDayPeriod))
```

### Why it double-subtracts

1. **`_workDaysInRequest` is already a Monday-to-Friday count.** The weekday `If` chain above it (lines 232-250) and `_numFullWeeks * 5` never count Saturdays or Sundays (Power Fx `Weekday()` returns 1 for Sunday and 7 for Saturday, and the chain treats 1 and 7 as non-working).
2. **`_holidaysInRequest` counts every holiday row in the date range, whatever day of the week it is.** `HolidaysCollection` (built in `App.fx.yaml` lines 142-147) keeps every Holiday row with `Working Day?` = No, and `CountIf` on line 252 only tests the date range.
3. **Line 253 subtracts the second number from the first.** A holiday on a Saturday or Sunday was never inside `_workDaysInRequest`, so subtracting it removes a real working day from the request.

A second way to get the same wrong result: two Holiday rows with the same date (for example an organisation closure entered on a public holiday) are counted twice by `CountIf`.

**Worked example: Fri 14 Aug to Mon 17 Aug 2026** (Independence Day is Saturday 15 Aug).

- 4 calendar days, 0 full weeks, a 4-day part-week starting on Friday, so the chain gives 2 working days (Fri, Mon).
- One holiday row falls in the range (15 Aug), so `_holidaysInRequest` = 1.
- `_requestedDays` = 2 - 1 = **1**. The correct answer is **2**: both days are ordinary working days.

**Consequences.** The employee is charged too few days. The wrong number is stored in `Duration` (line 547), shown in the balance preview (line 101, `'Current Balance' - _requestedDays`), and used in the Max Consecutive Days checks (lines 399 and 509). A request for Saturday 15 Aug alone gives 0 - 1 = **-1**: the days label shows -1 and the balance preview goes *up* by 1, although the Next button stays disabled because line 509 blocks `_requestedDays <= 0`.

In 2026 three holidays fall on a weekend: Id-ul-Fitr (Sat 21 Mar), Independence Day (Sat 15 Aug) and Diwali (Sun 8 Nov). The DoPT memorandum the holiday list comes from also says no substitute holiday is given when a festival falls on a weekly off (para 3.2), which matches not deducting it.

### Corrected code

Replace lines 252 and 253 (keep 251 and 254 as they are):

```powerfx
Set(_holidaysInRequest,
    CountRows(
        Distinct(
            Filter(
                HolidaysCollection,
                'Holiday Date' >= LeaveStartDatePicker.SelectedDate,
                'Holiday Date' <= LeaveEndDatePicker.SelectedDate,
                Weekday('Holiday Date') >= 2,   // Monday = 2 ...
                Weekday('Holiday Date') <= 6    // ... Friday = 6 (default Weekday(): Sunday = 1, Saturday = 7)
            ),
            'Holiday Date'
        )
    )
);
Set(_requestedDays, Max(0, _workDaysInRequest - _holidaysInRequest));
```

What changed and why:

- **`Weekday(...) >= 2` and `<= 6`** counts only holidays that fall Monday to Friday, the same days `_workDaysInRequest` counts. It uses the same Sunday-first numbering the existing chain already relies on.
- **`Distinct(..., 'Holiday Date')`** counts each holiday date once, even if two rows share a date.
- **`Max(0, ...)`** stops the days label going negative.

If Studio flags `Distinct` on the date column, drop it and keep the other two changes; only the duplicate-row case then remains.

---

## BUG-002: a 5-day leftover part-week starting on a Wednesday is over-counted

Found while checking BUG-001: the weekday chain itself is wrong for exactly one case out of 42 (leftover days 0-6 by start weekday). In my random test the chain disagreed with a brute-force count in 410 of 20,000 ranges, about 2%.

### Current code

Lines 235-237:

```
235  _numFullDaysPartialWeek = 5,
236    If(_startWeekday = 2, Set(_numPartialWeekdays, 5), _startWeekday = 1 || _startWeekday = 3 || _startWeekday = 4, Set(_numPartialWeekdays, 4), Set(_numPartialWeekdays, 3)
237    ),
```

### Why it is wrong

With 5 leftover days: starting Monday gives 5 weekdays; starting Sunday or Tuesday gives 4; starting Wednesday through Saturday gives 3. Line 236 lists `_startWeekday = 4` (Wednesday) with Sunday and Tuesday, so Wednesday returns 4 instead of 3. This over-counts, the opposite direction to BUG-001.

**Example: Wed 1 Jul to Sun 5 Jul 2026.** Real working days are Wed, Thu and Fri = **3**. The chain returns 4, and no holiday is involved. The same happens for any range starting on a Wednesday whose length is 5, 12, 19, ... days.

### Corrected code

Minimal fix: delete `|| _startWeekday = 4` on line 236.

```
236    If(_startWeekday = 2, Set(_numPartialWeekdays, 5), _startWeekday = 1 || _startWeekday = 3, Set(_numPartialWeekdays, 4), Set(_numPartialWeekdays, 3)
```

I checked the other 41 combinations against brute force; only this one differs.

Optional simplification (not required): replace the whole chain (lines 227-251) with a direct count. The logic matched brute force in all 20,000 test ranges, but this is the piece most likely to need syntax adjustments in Studio:

```powerfx
Set(_inclusiveTotalDaysRequested, DateDiff(LeaveStartDatePicker.SelectedDate, LeaveEndDatePicker.SelectedDate, TimeUnit.Days) + 1);
Set(_workDaysInRequest,
    CountRows(
        Filter(
            Sequence(_inclusiveTotalDaysRequested, 0),
            Weekday(DateAdd(LeaveStartDatePicker.SelectedDate, Value, TimeUnit.Days)) >= 2,
            Weekday(DateAdd(LeaveStartDatePicker.SelectedDate, Value, TimeUnit.Days)) <= 6
        )
    )
);
```

---

## A second copy of this code

`apps_src/requestleave/Src/HomeScreen.fx.yaml` lines 381-411 contain another copy of the same calculation, including the same Wednesday pattern on line 393. It counts holidays with `CountIf(Holidays, StartDate >= ThisItem.StartDate, ...)`, but the Holidays table has a `Holiday Date` column, not `StartDate`, so this looks like leftover template code. Check in Studio whether it shows formula errors or is reachable; if it is live, apply the same two fixes there.

## Applying the fixes in Studio

1. Open the Request Leave app for editing.
2. Tree view, `LeaveDatesScreen`, `LeaveStartDatePicker`, property `OnSelect`.
3. In the `_numFullDaysPartialWeek = 5` branch, remove `|| _startWeekday = 4` (BUG-002).
4. Replace the `_holidaysInRequest` and `_requestedDays` statements with the corrected code (BUG-001). Leave the final `If(_requestedDays = 1 && drpDayPeriod.Visible, Select(drpDayPeriod))` line.
5. Save and publish. If you re-export the app later, re-unpack it so `apps_src/` matches.

## Test table

"Correct" is weekdays in the range minus holidays that fall on a weekday. Numbers for the Current and After columns come from the Python port with the holiday list in `data/indian-holidays-2026.csv`.

| # | Range (2026) | Correct | Current | After BUG-001 fix | After both fixes |
|---|---|---|---|---|---|
| 1 | Fri 14 Aug to Mon 17 Aug (Sat holiday) | 2 | 1 | 2 | 2 |
| 2 | Sat 15 Aug only | 0 | -1 | 0 | 0 (Next stays disabled) |
| 3 | Fri 6 Nov to Tue 10 Nov (Sun holiday, Diwali) | 3 | 2 | 3 | 3 |
| 4 | Fri 20 Mar to Mon 23 Mar (Sat holiday, Id-ul-Fitr) | 2 | 1 | 2 | 2 |
| 5 | Thu 1 Oct to Mon 5 Oct, Gandhi Jayanti (Fri 2 Oct) entered twice | 2 | 1 | 2 | 2 |
| 6 | Thu 1 Oct to Mon 5 Oct, one Gandhi Jayanti row (control) | 2 | 2 | 2 | 2 |
| 7 | Thu 2 Apr to Mon 6 Apr, Good Friday (control) | 2 | 2 | 2 | 2 |
| 8 | Wed 1 Jul to Sun 5 Jul, no holiday (BUG-002) | 3 | 4 | 4 | 3 |

Rows 1, 2, 5 and 8 correspond to test cases TC-28, TC-29 and TC-34 in [test-cases.md](test-cases.md); rows 6 and 7 are the controls that must not change.
