<script lang="ts">
	import { onMount } from 'svelte';

	type ImperialEra = {
		startDate: string;
		name: string;
		koreanName: string;
		baseYear: number;
	};

	type ImperialCalendarConfig = {
		months: string[];
		eras: ImperialEra[];
	};

	type BovertSeason = {
		name: string;
		koreanName: string;
		lastDayName: string;
		koreanLastDayName: string;
	};

	type BovertCalendarConfig = {
		epochYear: number;
		yearNameOffset: number;
		yearStartOffsetDays: number;
		daysPerSeason: number;
		leapDayThreshold: number;
		yearNames: string[];
		koreanYearNames: string[];
		seasons: BovertSeason[];
		yearEndDayName: string;
		koreanYearEndDayName: string;
	};

	const DEFAULT_DATE_OFFSET = 196;
	const MIN_DATE_OFFSET = -730_119; // 0001-01-01
	const MAX_DATE_OFFSET = 2_921_939; // 9999-12-31

	let today = $state(new Date('2000-07-15'));
	let imperialCalendar = $state<ImperialCalendarConfig | null>(null);
	let imperialCalendarError = $state('');
	let bovertCalendar = $state<BovertCalendarConfig | null>(null);
	let bovertCalendarError = $state('');
	let numericEditor = $state<{
		kind: 'year' | 'month' | 'day';
		label: string;
		minimum: number;
		maximum: number;
	} | null>(null);
	let numericEditorValue = $state('');
	let imperialParts = $derived(
		imperialCalendar ? getImperialDateParts(today, imperialCalendar) : null
	);
	let bovertParts = $derived(bovertCalendar ? getBovertDateParts(today, bovertCalendar) : null);

	function isImperialCalendarConfig(value: unknown): value is ImperialCalendarConfig {
		if (typeof value !== 'object' || value === null) return false;

		const config = value as Record<string, unknown>;
		if (
			!Array.isArray(config.months) ||
			config.months.length !== 12 ||
			!config.months.every((month) => typeof month === 'string' && month.length > 0) ||
			!Array.isArray(config.eras) ||
			config.eras.length === 0
		) {
			return false;
		}

		return config.eras.every((era) => {
			if (typeof era !== 'object' || era === null) return false;
			const item = era as Record<string, unknown>;
			return (
				typeof item.startDate === 'string' &&
				/^\d{4}-\d{2}-\d{2}$/.test(item.startDate) &&
				typeof item.name === 'string' &&
				item.name.length > 0 &&
				typeof item.koreanName === 'string' &&
				item.koreanName.length > 0 &&
				Number.isSafeInteger(item.baseYear)
			);
		});
	}

	function isBovertCalendarConfig(value: unknown): value is BovertCalendarConfig {
		if (typeof value !== 'object' || value === null) return false;

		const config = value as Record<string, unknown>;
		const yearNamesAreValid =
			Array.isArray(config.yearNames) &&
			config.yearNames.length > 0 &&
			config.yearNames.every((name) => typeof name === 'string' && name.length > 0);
		const koreanYearNamesAreValid =
			Array.isArray(config.koreanYearNames) &&
			config.koreanYearNames.length === (config.yearNames as unknown[])?.length &&
			config.koreanYearNames.every((name) => typeof name === 'string' && name.length > 0);
		const seasonsAreValid =
			Array.isArray(config.seasons) &&
			config.seasons.length > 0 &&
			config.seasons.every((season) => {
				if (typeof season !== 'object' || season === null) return false;
				const item = season as Record<string, unknown>;
				return ['name', 'koreanName', 'lastDayName', 'koreanLastDayName'].every(
					(key) => typeof item[key] === 'string' && item[key].length > 0
				);
			});

		return (
			['epochYear', 'yearNameOffset', 'yearStartOffsetDays', 'leapDayThreshold'].every((key) =>
				Number.isSafeInteger(config[key])
			) &&
			Number.isSafeInteger(config.daysPerSeason) &&
			(config.daysPerSeason as number) > 1 &&
			yearNamesAreValid &&
			koreanYearNamesAreValid &&
			seasonsAreValid &&
			typeof config.yearEndDayName === 'string' &&
			config.yearEndDayName.length > 0 &&
			typeof config.koreanYearEndDayName === 'string' &&
			config.koreanYearEndDayName.length > 0
		);
	}

	function getImperialDateParts(d: Date, config: ImperialCalendarConfig) {
		const day = d.getDate();
		const month = d.getMonth() + 1; // Months are zero-based in JavaScript

		// date
		if (month < 1 || month > 12) {
			throw new Error('Month must be between 1 and 12');
		}

		const dateKey = `${String(d.getFullYear()).padStart(4, '0')}-${String(month).padStart(2, '0')}-${String(day).padStart(2, '0')}`;
		const era = [...config.eras]
			.sort((a, b) => b.startDate.localeCompare(a.startDate))
			.find((candidate) => dateKey >= candidate.startDate);

		if (!era) throw new Error('No imperial era is configured for this date');

		const year = d.getFullYear() - era.baseYear;

		return { era, year, month, day, monthName: config.months[month - 1] };
	}

	function getBovertDateParts(d: Date, config: BovertCalendarConfig) {
		let year = d.getFullYear();

		let days = Math.floor(
			(d.getTime() - new Date(`${d.getFullYear()}-01-01`).getTime()) / (1000 * 60 * 60 * 24) -
				config.yearStartOffsetDays
		);

		if (
			!((d.getFullYear() % 4 === 0 && d.getFullYear() % 100 !== 0) || d.getFullYear() % 400 === 0)
		) {
			days++;
		}
		if (days < 0) {
			days += 365;
			year--;
		}

		if (((year + 1) % 4 === 0 && (year + 1) % 100 !== 0) || (year + 1) % 400 === 0) {
			if (days >= config.leapDayThreshold) {
				days++;
			}
		}

		const seasonIndex = Math.min(
			Math.floor(days / config.daysPerSeason),
			config.seasons.length - 1
		);
		const season = config.seasons[seasonIndex];
		const dayIndex = days % config.daysPerSeason;
		const names = config.yearNames;
		const yearIndex =
			(((year - config.epochYear + config.yearNameOffset) % names.length) + names.length) %
			names.length;

		return {
			year,
			days,
			yearIndex,
			seasonIndex,
			season,
			dayIndex,
			isSeasonEnd: dayIndex === config.daysPerSeason - 1,
			isYearEnd: days === 365
		};
	}

	function makeLocalDate(year: number, month: number, day: number): Date {
		const date = new Date(2000, month - 1, 1);
		date.setFullYear(year, month - 1, day);
		return date;
	}

	function updateToday(date: Date) {
		const timezoneOffset = date.getTimezoneOffset() * 60000;
		today = new Date(date.getTime() - timezoneOffset);
	}

	function updateImperialEra(startDate: string) {
		if (!imperialCalendar || !imperialParts) return;
		const era = imperialCalendar.eras.find((item) => item.startDate === startDate);
		if (!era) return;

		const targetYear = era.baseYear + imperialParts.year;
		const maxDay = new Date(targetYear, imperialParts.month, 0).getDate();
		let date = makeLocalDate(targetYear, imperialParts.month, Math.min(imperialParts.day, maxDay));
		const sortedEras = [...imperialCalendar.eras].sort((a, b) =>
			a.startDate.localeCompare(b.startDate)
		);
		const eraIndex = sortedEras.findIndex((item) => item.startDate === era.startDate);
		const start = makeLocalDate(
			...(era.startDate.split('-').map(Number) as [number, number, number])
		);
		const nextEra = sortedEras[eraIndex + 1];
		if (date < start) date = start;
		if (nextEra) {
			const end = makeLocalDate(
				...(nextEra.startDate.split('-').map(Number) as [number, number, number])
			);
			end.setDate(end.getDate() - 1);
			if (date > end) date = end;
		}
		updateToday(date);
	}

	function openNumericEditor(
		kind: 'year' | 'month' | 'day',
		label: string,
		current: number,
		minimum: number,
		maximum: number
	) {
		numericEditor = { kind, label, minimum, maximum };
		numericEditorValue = String(current);
	}

	function focusNumericInput(node: HTMLInputElement) {
		requestAnimationFrame(() => {
			node.focus();
			node.select();
		});
	}

	function handleWindowKeydown(event: KeyboardEvent) {
		if (numericEditor && event.key === 'Escape') numericEditor = null;
	}

	function updateImperialYear() {
		if (!imperialParts) return;
		openNumericEditor(
			'year',
			'연도',
			imperialParts.year,
			1 - imperialParts.era.baseYear,
			9999 - imperialParts.era.baseYear
		);
	}

	function updateImperialMonth() {
		if (!imperialParts) return;
		openNumericEditor('month', '월', imperialParts.month, 1, 12);
	}

	function updateImperialDay() {
		if (!imperialParts) return;
		const maxDay = new Date(today.getFullYear(), imperialParts.month, 0).getDate();
		openNumericEditor('day', '일', imperialParts.day, 1, maxDay);
	}

	function submitNumericEditor(event: SubmitEvent) {
		event.preventDefault();
		const rawValue = String(numericEditorValue).trim();
		if (!numericEditor || !imperialParts || !/^-?\d+$/.test(rawValue)) return;

		const value = Number(rawValue);
		if (
			!Number.isSafeInteger(value) ||
			value < numericEditor.minimum ||
			value > numericEditor.maximum
		) {
			return;
		}

		if (numericEditor.kind === 'year') {
			const targetYear = imperialParts.era.baseYear + value;
			const maxDay = new Date(targetYear, imperialParts.month, 0).getDate();
			updateToday(
				makeLocalDate(targetYear, imperialParts.month, Math.min(imperialParts.day, maxDay))
			);
		} else if (numericEditor.kind === 'month') {
			const maxDay = new Date(today.getFullYear(), value, 0).getDate();
			updateToday(makeLocalDate(today.getFullYear(), value, Math.min(imperialParts.day, maxDay)));
		} else {
			updateToday(makeLocalDate(today.getFullYear(), imperialParts.month, value));
		}
		numericEditor = null;
	}

	function findBovertDate(year: number, days: number, config: BovertCalendarConfig) {
		const candidate = makeLocalDate(year, 1, 1);
		for (let index = 0; index < 732; index++) {
			const parts = getBovertDateParts(candidate, config);
			if (parts.year === year && parts.days === days) return candidate;
			candidate.setDate(candidate.getDate() + 1);
		}
		return null;
	}

	function updateBovertDate(year: number, days: number) {
		if (!bovertCalendar) return;
		const date = findBovertDate(year, days, bovertCalendar);
		if (date) updateToday(date);
	}

	function updateBovertYearName(index: number) {
		if (!bovertCalendar || !bovertParts) return;
		const cycle = bovertCalendar.yearNames.length;
		const baseYear = bovertCalendar.epochYear - bovertCalendar.yearNameOffset + index;
		const year = baseYear + Math.round((bovertParts.year - baseYear) / cycle) * cycle;
		updateBovertDate(year, bovertParts.days);
	}

	function updateBovertSeason(index: number) {
		if (!bovertCalendar || !bovertParts) return;
		const dayIndex = bovertParts.isYearEnd ? 0 : bovertParts.dayIndex;
		updateBovertDate(bovertParts.year, index * bovertCalendar.daysPerSeason + dayIndex);
	}

	function updateBovertDay(days: number) {
		if (bovertParts) updateBovertDate(bovertParts.year, days);
	}

	function toPreviousDay() {
		today = new Date(today.getFullYear(), today.getMonth(), today.getDate() - 1);
		const timezoneOffset = today.getTimezoneOffset() * 60000; // in milliseconds
		today = new Date(today.getTime() - timezoneOffset);
	}

	function toNextDay() {
		today = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 1);
		const timezoneOffset = today.getTimezoneOffset() * 60000; // in milliseconds
		today = new Date(today.getTime() - timezoneOffset);
	}

	function toPreviousMonth() {
		today = new Date(today.getFullYear(), today.getMonth() - 1, today.getDate());
		const timezoneOffset = today.getTimezoneOffset() * 60000; // in milliseconds
		today = new Date(today.getTime() - timezoneOffset);
	}

	function toNextMonth() {
		today = new Date(today.getFullYear(), today.getMonth() + 1, today.getDate());
		const timezoneOffset = today.getTimezoneOffset() * 60000; // in milliseconds
		today = new Date(today.getTime() - timezoneOffset);
	}

	function toPreviousYear() {
		today = new Date(today.getFullYear() - 1, today.getMonth(), today.getDate());
		const timezoneOffset = today.getTimezoneOffset() * 60000; // in milliseconds
		today = new Date(today.getTime() - timezoneOffset);
	}

	function toNextYear() {
		today = new Date(today.getFullYear() + 1, today.getMonth(), today.getDate());
		const timezoneOffset = today.getTimezoneOffset() * 60000; // in milliseconds
		today = new Date(today.getTime() - timezoneOffset);
	}

	function toPreviousCycle() {
		today = new Date(today.getFullYear() - 72, today.getMonth(), today.getDate());
		const timezoneOffset = today.getTimezoneOffset() * 60000;
		today = new Date(today.getTime() - timezoneOffset);
	}

	function toNextCycle() {
		today = new Date(today.getFullYear() + 72, today.getMonth(), today.getDate());
		const timezoneOffset = today.getTimezoneOffset() * 60000;
		today = new Date(today.getTime() - timezoneOffset);
	}

	onMount(async () => {
		try {
			const response = await fetch('/zasokese-calendar.json', { cache: 'no-store' });
			if (!response.ok) throw new Error(`HTTP ${response.status}`);

			const config: unknown = await response.json();
			if (!isImperialCalendarConfig(config)) throw new Error('Invalid configuration');
			imperialCalendar = config;
		} catch (error) {
			console.error('Failed to load the imperial calendar configuration', error);
			imperialCalendarError = '제국력 설정을 불러오지 못했습니다.';
		}

		try {
			const response = await fetch('/bovert-calendar.json', { cache: 'no-store' });
			if (!response.ok) throw new Error(`HTTP ${response.status}`);

			const config: unknown = await response.json();
			if (!isBovertCalendarConfig(config)) throw new Error('Invalid configuration');
			bovertCalendar = config;
		} catch (error) {
			console.error('Failed to load the Bovert calendar configuration', error);
			bovertCalendarError = '보베르타력 설정을 불러오지 못했습니다.';
		}

		const rawDate = new URLSearchParams(window.location.search).get('date');
		const parsedDate = rawDate !== null && /^-?\d{1,7}$/.test(rawDate) ? Number(rawDate) : NaN;
		const dateOffset =
			Number.isSafeInteger(parsedDate) &&
			parsedDate >= MIN_DATE_OFFSET &&
			parsedDate <= MAX_DATE_OFFSET
				? parsedDate
				: DEFAULT_DATE_OFFSET;

		today = new Date('2000-01-01');
		today.setDate(today.getDate() + dateOffset);
		const timezoneOffset = today.getTimezoneOffset() * 60000; // in milliseconds
		today = new Date(today.getTime() - timezoneOffset);
	});
</script>

<svelte:window onkeydown={handleWindowKeydown} />

<div
	style="display: flex; justify-content: center; align-items: center; height: 100vh; flex-direction: column;"
>
	<div
		style="max-width: 1280px; text-align: center; gap: 32px; display: flex; justify-content: center; align-items: center;"
	>
		<div>
			<div style="font-size: 12px;">자소크력</div>
			{#if imperialCalendar && imperialParts}
				<div style="font-size: 32px;">
					<select
						class="text-control"
						aria-label="자소크력 연호"
						value={imperialParts.era.startDate}
						onchange={(event) => updateImperialEra(event.currentTarget.value)}
					>
						{#each imperialCalendar.eras as era}
							<option value={era.startDate}>{era.name}</option>
						{/each}
					</select>
					<button type="button" class="text-control" onclick={updateImperialYear}
						>{imperialParts.year}</button
					>,
					<button type="button" class="text-control" onclick={updateImperialDay}
						>{imperialParts.day}</button
					>
					<button type="button" class="text-control" onclick={updateImperialMonth}
						>{imperialParts.monthName}</button
					>
				</div>
				<div>
					<select
						class="text-control"
						aria-label="자소크력 연호"
						value={imperialParts.era.startDate}
						onchange={(event) => updateImperialEra(event.currentTarget.value)}
					>
						{#each imperialCalendar.eras as era}
							<option value={era.startDate}>{era.koreanName}</option>
						{/each}
					</select>
					<button type="button" class="text-control" onclick={updateImperialYear}
						>{imperialParts.year === 0 ? '원년' : `${imperialParts.year}년`}</button
					>
					<button type="button" class="text-control" onclick={updateImperialMonth}
						>{imperialParts.month}월</button
					>
					<button type="button" class="text-control" onclick={updateImperialDay}
						>{imperialParts.day}일</button
					>
				</div>
			{:else if imperialCalendarError}
				<div role="alert">{imperialCalendarError}</div>
			{:else}
				<div>설정 불러오는 중…</div>
			{/if}
		</div>
		<div>
			<div style="font-size: 12px;">보베르타력</div>
			{#if bovertCalendar && bovertParts}
				<div style="font-size: 32px;">
					<select
						class="text-control"
						aria-label="보베르타력 일명"
						value={bovertParts.days}
						onchange={(event) => updateBovertDay(Number(event.currentTarget.value))}
					>
						{#each bovertCalendar.yearNames.slice(0, bovertCalendar.daysPerSeason - 1) as name, index}
							<option value={bovertParts.seasonIndex * bovertCalendar.daysPerSeason + index}
								>{name}</option
							>
						{/each}
						{#each bovertCalendar.seasons as season, index}
							<option value={(index + 1) * bovertCalendar.daysPerSeason - 1}
								>{season.lastDayName}</option
							>
						{/each}
						<option value={365}>{bovertCalendar.yearEndDayName}</option>
					</select>
					{#if !bovertParts.isSeasonEnd && !bovertParts.isYearEnd}
						<select
							class="text-control"
							aria-label="보베르타력 계절"
							value={bovertParts.seasonIndex}
							onchange={(event) => updateBovertSeason(Number(event.currentTarget.value))}
						>
							{#each bovertCalendar.seasons as season, index}
								<option value={index}>{season.name}</option>
							{/each}
						</select>
					{/if}
					<select
						class="text-control"
						aria-label="보베르타력 연명"
						value={bovertParts.yearIndex}
						onchange={(event) => updateBovertYearName(Number(event.currentTarget.value))}
					>
						{#each bovertCalendar.yearNames as name, index}
							<option value={index}>{name}</option>
						{/each}
					</select>
				</div>
				<div>
					<select
						class="text-control"
						aria-label="보베르타력 연명"
						value={bovertParts.yearIndex}
						onchange={(event) => updateBovertYearName(Number(event.currentTarget.value))}
					>
						{#each bovertCalendar.koreanYearNames as name, index}
							<option value={index}>{name}</option>
						{/each}
					</select>
					{#if !bovertParts.isSeasonEnd && !bovertParts.isYearEnd}
						<select
							class="text-control"
							aria-label="보베르타력 계절"
							value={bovertParts.seasonIndex}
							onchange={(event) => updateBovertSeason(Number(event.currentTarget.value))}
						>
							{#each bovertCalendar.seasons as season, index}
								<option value={index}>{season.koreanName}</option>
							{/each}
						</select>
					{/if}
					<select
						class="text-control"
						aria-label="보베르타력 일명"
						value={bovertParts.days}
						onchange={(event) => updateBovertDay(Number(event.currentTarget.value))}
					>
						{#each bovertCalendar.koreanYearNames.slice(0, bovertCalendar.daysPerSeason - 1) as name, index}
							<option value={bovertParts.seasonIndex * bovertCalendar.daysPerSeason + index}
								>{name}</option
							>
						{/each}
						{#each bovertCalendar.seasons as season, index}
							<option value={(index + 1) * bovertCalendar.daysPerSeason - 1}
								>{season.koreanLastDayName}</option
							>
						{/each}
						<option value={365}>{bovertCalendar.koreanYearEndDayName}</option>
					</select>
				</div>
			{:else if bovertCalendarError}
				<div role="alert">{bovertCalendarError}</div>
			{:else}
				<div>설정 불러오는 중…</div>
			{/if}
		</div>
	</div>
	<div
		style="display: flex; justify-content: center; align-items: center; gap: 24px; margin-top: 16px;"
	>
		<button type="button" class="clickable" onclick={toPreviousCycle}>← 72년</button>
		<button type="button" class="clickable" onclick={toPreviousYear}>← 년</button>
		<button type="button" class="clickable" onclick={toPreviousMonth}>← 월</button>
		<button type="button" class="clickable" onclick={toPreviousDay}>← 일</button>
		<button type="button" class="clickable" onclick={toNextDay}>일 →</button>
		<button type="button" class="clickable" onclick={toNextMonth}>월 →</button>
		<button type="button" class="clickable" onclick={toNextYear}>년 →</button>
		<button type="button" class="clickable" onclick={toNextCycle}>72년 →</button>
	</div>
</div>

{#if numericEditor}
	<div
		class="editor-overlay"
		role="dialog"
		aria-modal="true"
		aria-labelledby="numeric-editor-label"
	>
		<form class="editor-panel" onsubmit={submitNumericEditor}>
			<label id="numeric-editor-label" for="numeric-editor-input">{numericEditor.label}</label>
			<input
				id="numeric-editor-input"
				type="number"
				min={numericEditor.minimum}
				max={numericEditor.maximum}
				step="1"
				bind:value={numericEditorValue}
				use:focusNumericInput
			/>
			<div class="editor-actions">
				<button type="button" onclick={() => (numericEditor = null)}>취소</button>
				<button type="submit">확인</button>
			</div>
		</form>
	</div>
{/if}

<style>
	.text-control {
		all: unset;
		field-sizing: content;
		cursor: pointer;
	}

	.text-control:focus-visible {
		outline: 1px dotted currentColor;
		outline-offset: 2px;
	}

	.editor-overlay {
		position: fixed;
		inset: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		background: rgb(0 0 0 / 20%);
	}

	.editor-panel {
		display: flex;
		min-width: 200px;
		flex-direction: column;
		gap: 12px;
		padding: 20px;
		background: white;
		border: 1px solid #ddd;
		border-radius: 8px;
		box-shadow: 0 12px 40px rgb(0 0 0 / 25%);
	}

	.editor-panel input {
		box-sizing: border-box;
		width: 100%;
		padding: 8px;
		border: none;
		font: inherit;
		border-radius: 4px;
		transition: background-color 0.3s;
		font-family: 'Noto Serif KR', serif;
	}

	.editor-panel input:focus {
		background-color: #f0f0f0;
		outline: none;
		border: none;
	}

	.editor-actions {
		display: flex;
		justify-content: flex-end;
		gap: 8px;
	}

	.editor-actions button {
		padding: 6px 12px;
		border: 0;
		border-radius: 3px;
		background: white;
		font: inherit;
		cursor: pointer;
	}

	.clickable {
		cursor: pointer;
		padding: 8px 16px;
		border: 0;
		background-color: #f0f0f0;
		border-radius: 4px;
		font: inherit;
		transition: background-color 0.3s;
	}

	.clickable:hover {
		background-color: #e0e0e0;
	}
</style>
