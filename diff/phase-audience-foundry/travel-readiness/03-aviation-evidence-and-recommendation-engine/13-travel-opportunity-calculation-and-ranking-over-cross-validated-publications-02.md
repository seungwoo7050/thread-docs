## `feat(search): calculate eligible travel opportunities`

diff --git a/travel_windows/search.py b/travel_windows/search.py
new file mode 100644
index 0000000..3403bc0
--- /dev/null
+++ b/travel_windows/search.py
@@ -0,0 +1,626 @@
+from __future__ import annotations
+
+from dataclasses import dataclass
+from datetime import date, datetime, time, timedelta
+from typing import Callable
+from uuid import UUID
+from zoneinfo import ZoneInfo, ZoneInfoNotFoundError
+
+from django.db.models import F
+from django.utils import timezone
+
+from entry_requirements.models import PASSPORT_POLICY_ID
+from reviews.models import (
+    PublicationRevision,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+)
+from travel_windows.models import (
+    FLIGHT_POINTER_ID,
+    FlightPublication,
+    FlightPublicationDuration,
+    FlightSchedule,
+    FlightScheduleRevision,
+    PublishedFlightSchedule,
+    RouteDurationRevision,
+)
+
+
+SEOUL = ZoneInfo("Asia/Seoul")
+MINIMUM_LOCAL_STAY_MINUTES = 40 * 60
+MAXIMUM_CITY_RESULTS = 6
+MAXIMUM_ITINERARIES_PER_CITY = 2
+
+FLIGHT_READY = "ready"
+FLIGHT_EMPTY = "empty"
+FLIGHT_UNAVAILABLE = "unavailable"
+FLIGHT_STALE = "stale"
+FLIGHT_SERVER_ERROR = "server-error"
+
+_CALCULATION_NOTICE = (
+    "공식 인천 출발·도착 시각에 검수된 노선별 비행시간을 더하고 빼서 계산한 "
+    "예상값입니다. 공항 이동, 입출국 수속과 시내 이동 시간은 차감하지 않았습니다."
+)
+
+
+class _FlightEvidenceUnavailable(RuntimeError):
+    pass
+
+
+@dataclass(frozen=True, slots=True)
+class _AirportEvidence:
+    id: UUID
+    country_id: UUID
+    city_code: str
+    city_name: str
+    country_name: str
+    airport_name: str
+    airport_code: str
+    iana_timezone: str
+
+
+@dataclass(frozen=True, slots=True)
+class _ScheduleEvidence:
+    direction: str
+    carrier_name: str
+    flight_number: str
+    master_flight_number: str
+    airport: _AirportEvidence
+    icn_event_time: time
+    valid_from: date
+    valid_until: date
+    weekday_mask: str
+
+
+@dataclass(frozen=True, slots=True)
+class _DurationEvidence:
+    airport_id: UUID
+    outbound_minutes: int
+    inbound_minutes: int
+    reference_date: date
+
+
+@dataclass(frozen=True, slots=True)
+class _CurrentFlightEvidence:
+    generation: int
+    source_revision: str
+    source_locator: str
+    source_attribution: str
+    source_checked_at: datetime
+    schedule_source_date: date
+    schedules: tuple[_ScheduleEvidence, ...]
+    durations: tuple[_DurationEvidence, ...]
+
+
+@dataclass(frozen=True, slots=True)
+class _ScheduledEvent:
+    schedule: _ScheduleEvidence
+    icn_event_at: datetime
+
+
+@dataclass(frozen=True, slots=True)
+class _Candidate:
+    airport: _AirportEvidence
+    outbound: _ScheduledEvent
+    inbound: _ScheduledEvent
+    destination_arrival_at: datetime
+    destination_departure_at: datetime
+    stay_minutes: int
+    return_slack_seconds: int
+    duration_reference_date: date
+
+
+PublicationCardLoader = Callable[[UUID], dict[str, dict[str, object]]]
+
+
+def _load_eligible_country_ids() -> frozenset[UUID]:
+    entry_country_ids = set(
+        PublishedEntryFacts.objects.filter(
+            passport_policy_id=PASSPORT_POLICY_ID,
+            current_publication__isnull=False,
+            current_publication__state=PublicationRevision.State.PUBLISHED,
+            current_publication__scope_country_id=F("country_id"),
+        ).values_list("country_id", flat=True)
+    )
+    warning_country_ids = set(
+        PublishedTravelWarning.objects.filter(
+            current_publication__isnull=False,
+            current_publication__state=PublicationRevision.State.PUBLISHED,
+            current_publication__scope_country_id=F("country_id"),
+        ).values_list("country_id", flat=True)
+    )
+    return frozenset(entry_country_ids & warning_country_ids)
+
+
+def _load_current_flight_evidence() -> _CurrentFlightEvidence | None:
+    pointer = (
+        PublishedFlightSchedule.objects.select_related(
+            "current_publication__schedule_revision"
+        )
+        .filter(id=FLIGHT_POINTER_ID)
+        .first()
+    )
+    if pointer is None or pointer.current_publication_id is None:
+        return None
+
+    publication = pointer.current_publication
+    revision = publication.schedule_revision
+    if (
+        publication.state != FlightPublication.State.PUBLISHED
+        or revision.state != FlightScheduleRevision.State.VALIDATED
+        or revision.completeness != FlightScheduleRevision.Completeness.COMPLETE
+        or timezone.is_naive(publication.source_checked_at)
+    ):
+        raise _FlightEvidenceUnavailable
+
+    rows = FlightSchedule.objects.filter(revision=revision).select_related(
+        "destination_airport__country"
+    )
+    schedules: list[_ScheduleEvidence] = []
+    for row in rows:
+        airport = row.destination_airport
+        if not airport.is_public:
+            continue
+        try:
+            ZoneInfo(airport.iana_timezone)
+        except ZoneInfoNotFoundError as exc:
+            raise _FlightEvidenceUnavailable from exc
+        schedules.append(
+            _ScheduleEvidence(
+                direction=row.direction,
+                carrier_name=row.carrier_name,
+                flight_number=row.flight_number,
+                master_flight_number=row.master_flight_number,
+                airport=_AirportEvidence(
+                    id=airport.id,
+                    country_id=airport.country_id,
+                    city_code=airport.city_code,
+                    city_name=airport.city_name_ko,
+                    country_name=airport.country.name_ko,
+                    airport_name=airport.name_ko,
+                    airport_code=airport.iata_code,
+                    iana_timezone=airport.iana_timezone,
+                ),
+                icn_event_time=row.icn_event_time,
+                valid_from=row.valid_from,
+                valid_until=row.valid_until,
+                weekday_mask=row.weekday_mask,
+            )
+        )
+
+    links = FlightPublicationDuration.objects.filter(
+        publication=publication
+    ).select_related("duration_revision")
+    durations: list[_DurationEvidence] = []
+    seen_airports: set[UUID] = set()
+    for link in links:
+        duration = link.duration_revision
+        if (
+            duration.destination_airport_id != link.destination_airport_id
+            or duration.state != RouteDurationRevision.State.VALIDATED
+            or link.destination_airport_id in seen_airports
+        ):
+            raise _FlightEvidenceUnavailable
+        seen_airports.add(link.destination_airport_id)
+        durations.append(
+            _DurationEvidence(
+                airport_id=link.destination_airport_id,
+                outbound_minutes=duration.outbound_minutes,
+                inbound_minutes=duration.inbound_minutes,
+                reference_date=duration.reference_date,
+            )
+        )
+
+    if not schedules or not durations:
+        raise _FlightEvidenceUnavailable
+    return _CurrentFlightEvidence(
+        generation=publication.generation,
+        source_revision=publication.source_revision,
+        source_locator=publication.source_locator,
+        source_attribution=publication.source_attribution,
+        source_checked_at=publication.source_checked_at,
+        schedule_source_date=revision.source_date,
+        schedules=tuple(schedules),
+        durations=tuple(durations),
+    )
+
+
+def _date_range(first: date, last: date):
+    current = first
+    while current <= last:
+        yield current
+        current += timedelta(days=1)
+
+
+def _schedule_events(
+    schedules: tuple[_ScheduleEvidence, ...],
+    *,
+    direction: str,
+    departure_at: datetime,
+    return_by: datetime,
+) -> tuple[_ScheduledEvent, ...]:
+    selected: dict[tuple[str, str, UUID, datetime], _ScheduledEvent] = {}
+    for schedule in schedules:
+        if schedule.direction != direction:
+            continue
+        for event_date in _date_range(
+            departure_at.astimezone(SEOUL).date(),
+            return_by.astimezone(SEOUL).date(),
+        ):
+            if not schedule.valid_from <= event_date <= schedule.valid_until:
+                continue
+            if schedule.weekday_mask[event_date.weekday()] != "1":
+                continue
+            event_at = datetime.combine(
+                event_date,
+                schedule.icn_event_time.replace(tzinfo=None),
+                tzinfo=SEOUL,
+            )
+            if not departure_at <= event_at <= return_by:
+                continue
+            event = _ScheduledEvent(schedule=schedule, icn_event_at=event_at)
+            key = (
+                direction,
+                schedule.master_flight_number,
+                schedule.airport.id,
+                event_at,
+            )
+            previous = selected.get(key)
+            if previous is None or _event_identity(event) < _event_identity(previous):
+                selected[key] = event
+    return tuple(selected.values())
+
+
+def _event_identity(event: _ScheduledEvent) -> tuple[bool, str, str]:
+    schedule = event.schedule
+    return (
+        schedule.flight_number != schedule.master_flight_number,
+        schedule.flight_number,
+        schedule.carrier_name,
+    )
+
+
+def _candidate_sort_key(candidate: _Candidate) -> tuple:
+    return (
+        -candidate.stay_minutes,
+        -candidate.return_slack_seconds,
+        -candidate.outbound.icn_event_at.timestamp(),
+        candidate.airport.city_code,
+        candidate.airport.airport_code,
+        candidate.outbound.schedule.master_flight_number,
+        candidate.inbound.schedule.master_flight_number,
+    )
+
+
+def _calculate_candidates(
+    evidence: _CurrentFlightEvidence,
+    *,
+    eligible_country_ids: frozenset[UUID],
+    departure_at: datetime,
+    return_by: datetime,
+) -> tuple[tuple[_Candidate, ...], ...]:
+    durations = {row.airport_id: row for row in evidence.durations}
+    schedules = tuple(
+        schedule
+        for schedule in evidence.schedules
+        if schedule.airport.country_id in eligible_country_ids
+        and schedule.airport.id in durations
+    )
+    outbound_events = _schedule_events(
+        schedules,
+        direction=FlightSchedule.Direction.OUTBOUND,
+        departure_at=departure_at,
+        return_by=return_by,
+    )
+    inbound_events = _schedule_events(
+        schedules,
+        direction=FlightSchedule.Direction.INBOUND,
+        departure_at=departure_at,
+        return_by=return_by,
+    )
+
+    candidates: list[_Candidate] = []
+    for outbound in outbound_events:
+        duration = durations[outbound.schedule.airport.id]
+        destination_zone = ZoneInfo(outbound.schedule.airport.iana_timezone)
+        destination_arrival_at = (
+            outbound.icn_event_at + timedelta(minutes=duration.outbound_minutes)
+        ).astimezone(destination_zone)
+        for inbound in inbound_events:
+            if inbound.schedule.airport.id != outbound.schedule.airport.id:
+                continue
+            destination_departure_at = (
+                inbound.icn_event_at - timedelta(minutes=duration.inbound_minutes)
+            ).astimezone(destination_zone)
+            stay_minutes = int(
+                (
+                    destination_departure_at - destination_arrival_at
+                ).total_seconds()
+                // 60
+            )
+            if stay_minutes < MINIMUM_LOCAL_STAY_MINUTES:
+                continue
+            candidates.append(
+                _Candidate(
+                    airport=outbound.schedule.airport,
+                    outbound=outbound,
+                    inbound=inbound,
+                    destination_arrival_at=destination_arrival_at,
+                    destination_departure_at=destination_departure_at,
+                    stay_minutes=stay_minutes,
+                    return_slack_seconds=int(
+                        (return_by - inbound.icn_event_at).total_seconds()
+                    ),
+                    duration_reference_date=duration.reference_date,
+                )
+            )
+
+    candidates.sort(key=_candidate_sort_key)
+    by_city: dict[str, list[_Candidate]] = {}
+    for candidate in candidates:
+        city_rows = by_city.setdefault(candidate.airport.city_code, [])
+        if len(city_rows) < MAXIMUM_ITINERARIES_PER_CITY:
+            city_rows.append(candidate)
+    ranked_cities = sorted(by_city.values(), key=lambda rows: _candidate_sort_key(rows[0]))
+    return tuple(tuple(rows) for rows in ranked_cities[:MAXIMUM_CITY_RESULTS])
+
+
+def _format_kst(value: datetime) -> str:
+    return value.astimezone(SEOUL).strftime("%Y.%m.%d %H:%M KST")
+
+
+def _format_local(value: datetime) -> str:
+    return value.strftime("%Y.%m.%d %H:%M 현지시각")
+
+
+def _format_stay(minutes: int) -> str:
+    hours, remainder = divmod(minutes, 60)
+    return f"{hours}시간" if remainder == 0 else f"{hours}시간 {remainder}분"
+
+
+def _schedule_context(
+    event: _ScheduledEvent,
+    *,
+    destination_event_at: datetime,
+) -> dict[str, str]:
+    return {
+        "carrier_name": event.schedule.carrier_name,
+        "flight_number": event.schedule.flight_number,
+        "icn_event_label": _format_kst(event.icn_event_at),
+        "estimated_destination_event_label": _format_local(destination_event_at),
+    }
+
+
+def _state_card(module: str, state: str) -> dict[str, object]:
+    is_entry = module == "entry"
+    status_labels = {
+        FLIGHT_EMPTY: "게시된 자료 없음",
+        FLIGHT_UNAVAILABLE: "현재 확인할 수 없음",
+        FLIGHT_STALE: "재확인 필요",
+        FLIGHT_SERVER_ERROR: "일시적 오류",
+    }
+    return {
+        "module": module,
+        "heading": "입국요건" if is_entry else "여행경보",
+        "state": state,
+        "status_label": status_labels.get(state, "현재 게시본"),
+        "message": (
+            "입국요건을 지금 읽을 수 없습니다. 여행경보는 별도로 확인해 주세요."
+            if is_entry
+            else "여행경보를 지금 읽을 수 없습니다. 입국요건은 별도로 확인해 주세요."
+        ),
+        "has_publication": False,
+    }
+
+
+def _basic_publication_card(
+    *,
+    module: str,
+    country_id: UUID,
+) -> dict[str, object]:
+    if module == "entry":
+        pointer_model = PublishedEntryFacts
+        heading = "입국요건"
+    else:
+        pointer_model = PublishedTravelWarning
+        heading = "여행경보"
+    row = (
+        pointer_model.objects.filter(
+            country_id=country_id,
+            current_publication__isnull=False,
+        )
+        .values(
+            "current_publication__generation",
+            "current_publication__published_at",
+            "current_publication__scope_country__name_ko",
+            "current_publication__source_revision",
+            "current_publication__source_owner_snapshot",
+            "current_publication__source_locator_snapshot",
+            "current_publication__attribution_text_snapshot",
+        )
+        .first()
+    )
+    if row is None:
+        return _state_card(module, FLIGHT_UNAVAILABLE)
+    return {
+        "module": module,
+        "heading": heading,
+        "state": FLIGHT_READY,
+        "status_label": "현재 게시본",
+        "message": "공식 출처에서 수집해 검수·게시한 자료입니다.",
+        "has_publication": True,
+        "country_name": row["current_publication__scope_country__name_ko"],
+        "generation": row["current_publication__generation"],
+        "published_at": _format_kst(row["current_publication__published_at"]),
+        "source_revision": row["current_publication__source_revision"],
+        "source_owner": row["current_publication__source_owner_snapshot"],
+        "source_locator": row["current_publication__source_locator_snapshot"],
+        "attribution": row["current_publication__attribution_text_snapshot"],
+        "checked_at": None,
+    }
+
+
+def _load_basic_publication_cards(country_id: UUID) -> dict[str, dict[str, object]]:
+    cards: dict[str, dict[str, object]] = {}
+    for module in ("entry", "warning"):
+        try:
+            cards[module] = _basic_publication_card(
+                module=module,
+                country_id=country_id,
+            )
+        except Exception:
+            cards[module] = _state_card(module, FLIGHT_SERVER_ERROR)
+    return cards
+
+
+def _itinerary_context(candidate: _Candidate) -> dict[str, object]:
+    return {
+        "estimated_local_stay_label": _format_stay(candidate.stay_minutes),
+        "outbound_schedule": _schedule_context(
+            candidate.outbound,
+            destination_event_at=candidate.destination_arrival_at,
+        ),
+        "inbound_schedule": _schedule_context(
+            candidate.inbound,
+            destination_event_at=candidate.destination_departure_at,
+        ),
+    }
+
+
+def _opportunities_context(
+    city_candidates: tuple[tuple[_Candidate, ...], ...],
+    *,
+    evidence: _CurrentFlightEvidence,
+    publication_card_loader: PublicationCardLoader,
+) -> list[dict[str, object]]:
+    opportunities: list[dict[str, object]] = []
+    card_cache: dict[UUID, dict[str, dict[str, object]]] = {}
+    for rank, candidates in enumerate(city_candidates, start=1):
+        primary = candidates[0]
+        cards = card_cache.get(primary.airport.country_id)
+        if cards is None:
+            try:
+                cards = publication_card_loader(primary.airport.country_id)
+            except Exception:
+                cards = {
+                    "entry": _state_card("entry", FLIGHT_SERVER_ERROR),
+                    "warning": _state_card("warning", FLIGHT_SERVER_ERROR),
+                }
+            card_cache[primary.airport.country_id] = cards
+        itinerary = _itinerary_context(primary)
+        opportunities.append(
+            {
+                "rank": rank,
+                "destination": {
+                    "city_name": primary.airport.city_name,
+                    "country_name": primary.airport.country_name,
+                    "airport_name": primary.airport.airport_name,
+                    "airport_code": primary.airport.airport_code,
+                },
+                "estimated_local_stay_minutes": primary.stay_minutes,
+                **itinerary,
+                "alternatives": [
+                    _itinerary_context(candidate) for candidate in candidates[1:]
+                ],
+                "calculation_basis": {
+                    "schedule_source_date": evidence.schedule_source_date.isoformat(),
+                    "duration_reference_date": primary.duration_reference_date.isoformat(),
+                    "notice": _CALCULATION_NOTICE,
+                },
+                "entry_card": cards.get(
+                    "entry", _state_card("entry", FLIGHT_UNAVAILABLE)
+                ),
+                "warning_card": cards.get(
+                    "warning", _state_card("warning", FLIGHT_UNAVAILABLE)
+                ),
+            }
+        )
+    return opportunities
+
+
+def _flight_context(
+    *,
+    state: str,
+    evidence: _CurrentFlightEvidence | None = None,
+    opportunities: list[dict[str, object]] | None = None,
+) -> dict[str, object]:
+    labels = {
+        FLIGHT_READY: "현재 게시본",
+        FLIGHT_EMPTY: "게시된 자료 없음",
+        FLIGHT_UNAVAILABLE: "현재 확인할 수 없음",
+        FLIGHT_STALE: "재확인 필요",
+        FLIGHT_SERVER_ERROR: "일시적 오류",
+    }
+    messages = {
+        FLIGHT_READY: (
+            "검수·게시된 정기운항편과 비행시간을 사용했습니다. 항공권 좌석과 실제 운항은 "
+            "항공사에서 다시 확인해 주세요."
+        ),
+        FLIGHT_EMPTY: "아직 게시된 정기운항편 자료가 없어 추천을 계산하지 못했습니다.",
+        FLIGHT_UNAVAILABLE: "게시된 운항 자료의 구성을 현재 확인할 수 없습니다.",
+        FLIGHT_STALE: "마지막 게시 운항 자료를 사용했습니다. 공식 출처에서 최신 일정을 재확인해 주세요.",
+        FLIGHT_SERVER_ERROR: "운항 자료를 지금 읽을 수 없습니다. 잠시 뒤 다시 시도해 주세요.",
+    }
+    return {
+        "flight_state": state,
+        "flight_status_label": labels[state],
+        "flight_message": messages[state],
+        "flight_checked_at": (
+            _format_kst(evidence.source_checked_at) if evidence is not None else None
+        ),
+        "flight_publication_revision": (
+            str(evidence.generation) if evidence is not None else None
+        ),
+        "flight_source_locator": (
+            evidence.source_locator if evidence is not None else None
+        ),
+        "flight_source_attribution": (
+            evidence.source_attribution if evidence is not None else None
+        ),
+        "opportunities": opportunities or [],
+    }
+
+
+def search_travel_opportunities(
+    *,
+    departure_at: datetime,
+    return_by: datetime,
+    publication_card_loader: PublicationCardLoader | None = None,
+) -> dict[str, object]:
+    """Read only current publications and calculate transient recommendations.
+
+    This function performs no transport calls and does not persist either input or
+    output. Its returned values are display-ready template context fragments.
+    """
+
+    if timezone.is_naive(departure_at) or timezone.is_naive(return_by):
+        raise ValueError("search window datetimes must be timezone-aware")
+    departure_at = departure_at.astimezone(SEOUL)
+    return_by = return_by.astimezone(SEOUL)
+    try:
+        evidence = _load_current_flight_evidence()
+        if evidence is None:
+            return _flight_context(state=FLIGHT_EMPTY)
+        eligible_country_ids = _load_eligible_country_ids()
+        city_candidates = _calculate_candidates(
+            evidence,
+            eligible_country_ids=eligible_country_ids,
+            departure_at=departure_at,
+            return_by=return_by,
+        )
+        opportunities = _opportunities_context(
+            city_candidates,
+            evidence=evidence,
+            publication_card_loader=(
+                publication_card_loader or _load_basic_publication_cards
+            ),
+        )
+        return _flight_context(
+            state=FLIGHT_READY,
+            evidence=evidence,
+            opportunities=opportunities,
+        )
+    except _FlightEvidenceUnavailable:
+        return _flight_context(state=FLIGHT_UNAVAILABLE)
+    except Exception:
+        return _flight_context(state=FLIGHT_SERVER_ERROR)
diff --git a/travel_windows/tests/test_search.py b/travel_windows/tests/test_search.py
new file mode 100644
index 0000000..fc7535d
--- /dev/null
+++ b/travel_windows/tests/test_search.py
@@ -0,0 +1,561 @@
+import json
+from datetime import date, datetime, time
+from unittest.mock import patch
+from zoneinfo import ZoneInfo
+
+from django.test import SimpleTestCase, TransactionTestCase
+
+from countries.models import SUPPORTED_COUNTRY_IDS, Country
+from operations.tests import migration_helpers as _migration_helpers  # noqa: F401
+from sources.management.commands.register_approved_sources import (
+    register_approved_sources,
+)
+from travel_windows.ingestion import (
+    FlightIngestionCode,
+    ingest_and_publish_flight_evidence,
+)
+from travel_windows.models import (
+    CURATED_AIRPORT_ROWS,
+    TIMEZONE_EVIDENCE_LOCATOR,
+    Airport,
+    FlightPublication,
+    PublishedFlightSchedule,
+)
+from travel_windows.search import (
+    FLIGHT_READY,
+    _AirportEvidence,
+    _CurrentFlightEvidence,
+    _DurationEvidence,
+    _ScheduleEvidence,
+    _load_eligible_country_ids,
+    search_travel_opportunities,
+)
+
+
+SEOUL = ZoneInfo("Asia/Seoul")
+SOURCE_DATE = date(2026, 8, 31)
+REFERENCE_DATE = date(2026, 8, 1)
+VALID_FROM = date(2026, 3, 29)
+VALID_UNTIL = date(2026, 10, 24)
+
+COUNTRY_NAMES = {
+    "JP": "일본",
+    "TW": "대만",
+    "HK": "홍콩",
+    "MO": "마카오",
+    "VN": "베트남",
+    "TH": "태국",
+}
+
+
+def _airport_evidence(iata_code):
+    row = next(row for row in CURATED_AIRPORT_ROWS if row[1] == iata_code)
+    airport_id, code, country, city_code, city_name, name, iana_timezone = row
+    return _AirportEvidence(
+        id=airport_id,
+        country_id=SUPPORTED_COUNTRY_IDS[country],
+        city_code=city_code,
+        city_name=city_name,
+        country_name=COUNTRY_NAMES[country],
+        airport_name=name,
+        airport_code=code,
+        iana_timezone=iana_timezone,
+    )
+
+
+def _weekday_mask(weekday):
+    values = ["0"] * 7
+    values[weekday] = "1"
+    return "".join(values)
+
+
+def _schedule(
+    airport,
+    *,
+    direction,
+    event_time,
+    flight_number,
+    master_flight_number=None,
+    carrier_name="대한항공",
+    weekday,
+):
+    return _ScheduleEvidence(
+        direction=direction,
+        carrier_name=carrier_name,
+        flight_number=flight_number,
+        master_flight_number=master_flight_number or flight_number,
+        airport=airport,
+        icn_event_time=event_time,
+        valid_from=VALID_FROM,
+        valid_until=VALID_UNTIL,
+        weekday_mask=_weekday_mask(weekday),
+    )
+
+
+def _duration(airport, outbound=120, inbound=120):
+    return _DurationEvidence(
+        airport_id=airport.id,
+        outbound_minutes=outbound,
+        inbound_minutes=inbound,
+        reference_date=REFERENCE_DATE,
+    )
+
+
+def _evidence(*, schedules, durations):
+    return _CurrentFlightEvidence(
+        generation=7,
+        source_revision="synthetic-search-v1",
+        source_locator="https://example.invalid/synthetic-schedule",
+        source_attribution="합성 검색 계약 fixture",
+        source_checked_at=datetime(2026, 8, 31, 12, tzinfo=SEOUL),
+        schedule_source_date=SOURCE_DATE,
+        schedules=tuple(schedules),
+        durations=tuple(durations),
+    )
+
+
+def _cards(_country_id):
+    return {
+        "entry": {"module": "entry", "state": "ready"},
+        "warning": {"module": "warning", "state": "ready"},
+    }
+
+
+def _search(evidence, eligible_country_ids, *, departure_at, return_by):
+    with (
+        patch(
+            "travel_windows.search._load_current_flight_evidence",
+            return_value=evidence,
+        ),
+        patch(
+            "travel_windows.search._load_eligible_country_ids",
+            return_value=frozenset(eligible_country_ids),
+        ),
+    ):
+        return search_travel_opportunities(
+            departure_at=departure_at,
+            return_by=return_by,
+            publication_card_loader=_cards,
+        )
+
+
+class TravelOpportunityCalculationTests(SimpleTestCase):
+    def test_40_hours_is_inclusive_and_codeshare_uses_the_master_flight(self):
+        nrt = _airport_evidence("NRT")
+        kix = _airport_evidence("KIX")
+        schedules = (
+            _schedule(
+                nrt,
+                direction="OUTBOUND",
+                event_time=time(9),
+                flight_number="OZ9701",
+                master_flight_number="KE701",
+                carrier_name="아시아나항공",
+                weekday=1,
+            ),
+            _schedule(
+                nrt,
+                direction="OUTBOUND",
+                event_time=time(9),
+                flight_number="KE701",
+                weekday=1,
+            ),
+            _schedule(
+                nrt,
+                direction="INBOUND",
+                event_time=time(5),
+                flight_number="KE704",
+                weekday=3,
+            ),
+            _schedule(
+                kix,
+                direction="OUTBOUND",
+                event_time=time(9),
+                flight_number="KE721",
+                weekday=1,
+            ),
+            _schedule(
+                kix,
+                direction="INBOUND",
+                event_time=time(4, 59),
+                flight_number="KE724",
+                weekday=3,
+            ),
+        )
+        result = _search(
+            _evidence(
+                schedules=schedules,
+                durations=(_duration(nrt), _duration(kix)),
+            ),
+            {SUPPORTED_COUNTRY_IDS["JP"]},
+            departure_at=datetime(2026, 9, 1, 8, tzinfo=SEOUL),
+            return_by=datetime(2026, 9, 3, 5, tzinfo=SEOUL),
+        )
+
+        self.assertEqual(result["flight_state"], FLIGHT_READY)
+        self.assertEqual(len(result["opportunities"]), 1)
+        opportunity = result["opportunities"][0]
+        self.assertEqual(opportunity["destination"]["airport_code"], "NRT")
+        self.assertEqual(opportunity["estimated_local_stay_minutes"], 2400)
+        self.assertEqual(opportunity["estimated_local_stay_label"], "40시간")
+        self.assertEqual(
+            opportunity["outbound_schedule"]["flight_number"], "KE701"
+        )
+        self.assertEqual(
+            opportunity["inbound_schedule"]["icn_event_label"],
+            "2026.09.03 05:00 KST",
+        )
+        self.assertEqual(
+            opportunity["inbound_schedule"][
+                "estimated_destination_event_label"
+            ],
+            "2026.09.03 03:00 현지시각",
+        )
+        self.assertEqual(opportunity["alternatives"], [])
+
+    def test_nrt_and_hnd_group_as_one_city_with_two_total_itineraries(self):
+        nrt = _airport_evidence("NRT")
+        hnd = _airport_evidence("HND")
+        schedules = (
+            _schedule(
+                nrt,
+                direction="OUTBOUND",
+                event_time=time(8),
+                flight_number="KE701",
+                weekday=1,
+            ),
+            _schedule(
+                nrt,
+                direction="INBOUND",
+                event_time=time(6),
+                flight_number="KE704",
+                weekday=3,
+            ),
+            _schedule(
+                nrt,
+                direction="INBOUND",
+                event_time=time(7),
+                flight_number="KE706",
+                weekday=3,
+            ),
+            _schedule(
+                hnd,
+                direction="OUTBOUND",
+                event_time=time(8, 30),
+                flight_number="KE719",
+                weekday=1,
+            ),
+            _schedule(
+                hnd,
+                direction="INBOUND",
+                event_time=time(6, 30),
+                flight_number="KE720",
+                weekday=3,
+            ),
+        )
+        result = _search(
+            _evidence(
+                schedules=schedules,
+                durations=(_duration(nrt), _duration(hnd)),
+            ),
+            {SUPPORTED_COUNTRY_IDS["JP"]},
+            departure_at=datetime(2026, 9, 1, 7, tzinfo=SEOUL),
+            return_by=datetime(2026, 9, 3, 8, tzinfo=SEOUL),
+        )
+
+        self.assertEqual(len(result["opportunities"]), 1)
+        opportunity = result["opportunities"][0]
+        self.assertEqual(opportunity["destination"]["city_name"], "도쿄")
+        self.assertEqual(opportunity["destination"]["airport_code"], "NRT")
+        self.assertEqual(
+            opportunity["inbound_schedule"]["flight_number"], "KE706"
+        )
+        self.assertEqual(len(opportunity["alternatives"]), 1)
+        self.assertEqual(
+            opportunity["alternatives"][0]["inbound_schedule"][
+                "flight_number"
+            ],
+            "KE704",
+        )
+
+    def test_results_are_limited_to_six_cities_with_city_code_tiebreak(self):
+        airport_codes = ("NRT", "KIX", "FUK", "CTS", "OKA", "TPE", "HKG")
+        airports = tuple(_airport_evidence(code) for code in airport_codes)
+        schedules = []
+        for index, airport in enumerate(airports, start=1):
+            schedules.extend(
+                (
+                    _schedule(
+                        airport,
+                        direction="OUTBOUND",
+                        event_time=time(9),
+                        flight_number=f"KE{700 + index}",
+                        weekday=1,
+                    ),
+                    _schedule(
+                        airport,
+                        direction="INBOUND",
+                        event_time=time(5),
+                        flight_number=f"KE{800 + index}",
+                        weekday=3,
+                    ),
+                )
+            )
+        eligible = {airport.country_id for airport in airports}
+        result = _search(
+            _evidence(
+                schedules=schedules,
+                durations=tuple(_duration(airport) for airport in airports),
+            ),
+            eligible,
+            departure_at=datetime(2026, 9, 1, 8, tzinfo=SEOUL),
+            return_by=datetime(2026, 9, 3, 6, tzinfo=SEOUL),
+        )
+
+        self.assertEqual(
+            [row["destination"]["airport_code"] for row in result["opportunities"]],
+            ["FUK", "HKG", "OKA", "KIX", "CTS", "TPE"],
+        )
+        self.assertEqual(
+            [row["rank"] for row in result["opportunities"]],
+            [1, 2, 3, 4, 5, 6],
+        )
+
+    def test_sorting_uses_stay_then_slack_then_departure_instant(self):
+        hkg = _airport_evidence("HKG")
+        nrt = _airport_evidence("NRT")
+        kix = _airport_evidence("KIX")
+        fuk = _airport_evidence("FUK")
+        definitions = (
+            # 41h stay: first even with the least return slack.
+            (hkg, time(8), time(5), 90, 90, "KE701", "KE801"),
+            # 40h, two hours slack, later departure.
+            (nrt, time(9), time(4), 90, 90, "KE702", "KE802"),
+            # 40h, two hours slack, earlier departure.
+            (kix, time(8), time(4), 120, 120, "KE703", "KE803"),
+            # 40h, one hour slack.
+            (fuk, time(9), time(5), 120, 120, "KE704", "KE804"),
+        )
+        schedules = []
+        durations = []
+        for (
+            airport,
+            outbound_at,
+            inbound_at,
+            out_minutes,
+            in_minutes,
+            out_no,
+            in_no,
+        ) in definitions:
+            schedules.extend(
+                (
+                    _schedule(
+                        airport,
+                        direction="OUTBOUND",
+                        event_time=outbound_at,
+                        flight_number=out_no,
+                        weekday=1,
+                    ),
+                    _schedule(
+                        airport,
+                        direction="INBOUND",
+                        event_time=inbound_at,
+                        flight_number=in_no,
+                        weekday=3,
+                    ),
+                )
+            )
+            durations.append(_duration(airport, out_minutes, in_minutes))
+        result = _search(
+            _evidence(schedules=schedules, durations=durations),
+            {airport.country_id for airport, *_rest in definitions},
+            departure_at=datetime(2026, 9, 1, 7, tzinfo=SEOUL),
+            return_by=datetime(2026, 9, 3, 6, tzinfo=SEOUL),
+        )
+
+        self.assertEqual(
+            [row["destination"]["airport_code"] for row in result["opportunities"]],
+            ["HKG", "NRT", "KIX", "FUK"],
+        )
+
+
+class PublicationEligibilityTests(SimpleTestCase):
+    def test_country_requires_both_current_entry_and_warning_publications(self):
+        japan = SUPPORTED_COUNTRY_IDS["JP"]
+        taiwan = SUPPORTED_COUNTRY_IDS["TW"]
+        vietnam = SUPPORTED_COUNTRY_IDS["VN"]
+        with (
+            patch("travel_windows.search.PublishedEntryFacts") as entry_model,
+            patch("travel_windows.search.PublishedTravelWarning") as warning_model,
+        ):
+            entry_model.objects.filter.return_value.values_list.return_value = (
+                japan,
+                taiwan,
+            )
+            warning_model.objects.filter.return_value.values_list.return_value = (
+                japan,
+                vietnam,
+            )
+
+            eligible = _load_eligible_country_ids()
+
+        self.assertEqual(eligible, frozenset({japan}))
+        entry_filters = entry_model.objects.filter.call_args.kwargs
+        warning_filters = warning_model.objects.filter.call_args.kwargs
+        self.assertFalse(entry_filters["current_publication__isnull"])
+        self.assertFalse(warning_filters["current_publication__isnull"])
+        self.assertEqual(
+            entry_filters["current_publication__state"], "PUBLISHED"
+        )
+        self.assertEqual(
+            warning_filters["current_publication__state"], "PUBLISHED"
+        )
+
+
+def _official_row(
+    *, airport_code, airport_name, flight_number, event_time, weekday
+):
+    weekday_fields = (
+        "ynMon",
+        "ynTue",
+        "ynWed",
+        "ynThu",
+        "ynFri",
+        "ynSat",
+        "ynSun",
+    )
+    row = {
+        "codeshare": "Master",
+        "masterFlightId": flight_number,
+        "flightId": flight_number,
+        "st": event_time,
+        "season": "S26",
+        "firstdate": "20260329",
+        "lastdate": "20261024",
+        "terminalId": "P02",
+        "airline": "대한항공",
+        "airlineCode": "KE",
+        "airport": airport_name,
+        "airportCode": airport_code,
+        "tmp1": "",
+        "tmp2": "",
+    }
+    row.update(
+        {
+            field: "Y" if index == weekday else "N"
+            for index, field in enumerate(weekday_fields)
+        }
+    )
+    return row
+
+
+def _official_page(rows):
+    return json.dumps(
+        {
+            "response": {
+                "header": {"resultCode": "00", "resultMsg": "NORMAL SERVICE."},
+                "body": {
+                    "items": {"item": rows},
+                    "pageNo": "1",
+                    "numOfRows": str(len(rows)),
+                    "totalCount": str(len(rows)),
+                },
+            }
+        },
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
+class CurrentFlightPublicationSearchTests(TransactionTestCase):
+    def setUp(self):
+        register_approved_sources(apply=True)
+        row = next(row for row in CURATED_AIRPORT_ROWS if row[1] == "NRT")
+        Airport.objects.get_or_create(
+            id=row[0],
+            defaults={
+                "iata_code": row[1],
+                "country": Country.objects.get(iso_alpha2=row[2]),
+                "city_code": row[3],
+                "city_name_ko": row[4],
+                "name_ko": row[5],
+                "iana_timezone": row[6],
+                "timezone_evidence_locator": TIMEZONE_EVIDENCE_LOCATOR,
+                "is_public": True,
+            },
+        )
+
+    def test_search_reads_current_publication_without_persisting_search_values(self):
+        nrt = Airport.objects.get(iata_code="NRT")
+        outcome = ingest_and_publish_flight_evidence(
+            departure_pages=(
+                _official_page(
+                    [
+                        _official_row(
+                            airport_code="NRT",
+                            airport_name=nrt.name_ko,
+                            flight_number="KE701",
+                            event_time="0900",
+                            weekday=1,
+                        )
+                    ]
+                ),
+            ),
+            arrival_pages=(
+                _official_page(
+                    [
+                        _official_row(
+                            airport_code="NRT",
+                            airport_name=nrt.name_ko,
+                            flight_number="KE704",
+                            event_time="0500",
+                            weekday=3,
+                        )
+                    ]
+                ),
+            ),
+            duration_csv=(
+                b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+                b"reference_locator\r\nNRT,120,120,2026-08-01,"
+                b"https://www.ulip.go.kr/\r\n"
+            ),
+            source_date=SOURCE_DATE,
+            published_by="synthetic-search-reviewer",
+        )
+        self.assertEqual(outcome.code, FlightIngestionCode.PUBLISHED)
+        before = (
+            PublishedFlightSchedule.objects.get().version,
+            FlightPublication.objects.count(),
+        )
+
+        without_policy_publications = search_travel_opportunities(
+            departure_at=datetime(2026, 9, 1, 8, tzinfo=SEOUL),
+            return_by=datetime(2026, 9, 3, 5, tzinfo=SEOUL),
+            publication_card_loader=_cards,
+        )
+        with patch(
+            "travel_windows.search._load_eligible_country_ids",
+            return_value=frozenset({SUPPORTED_COUNTRY_IDS["JP"]}),
+        ):
+            result = search_travel_opportunities(
+                departure_at=datetime(2026, 9, 1, 8, tzinfo=SEOUL),
+                return_by=datetime(2026, 9, 3, 5, tzinfo=SEOUL),
+                publication_card_loader=_cards,
+            )
+
+        self.assertEqual(without_policy_publications["flight_state"], FLIGHT_READY)
+        self.assertEqual(without_policy_publications["opportunities"], [])
+        self.assertEqual(result["flight_state"], FLIGHT_READY)
+        self.assertEqual(result["flight_publication_revision"], "1")
+        self.assertEqual(len(result["opportunities"]), 1)
+        self.assertEqual(
+            result["opportunities"][0]["estimated_local_stay_minutes"],
+            2400,
+        )
+        self.assertEqual(
+            (
+                PublishedFlightSchedule.objects.get().version,
+                FlightPublication.objects.count(),
+            ),
+            before,
+        )


