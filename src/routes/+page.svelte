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

	function formatZasokeseDate(
		d: Date,
		config: ImperialCalendarConfig,
		korean: boolean = false
	): string {
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

		const gengou = korean ? era.koreanName : era.name;
		const year = d.getFullYear() - era.baseYear;

		return korean
			? `${gengou} ${year === 0 ? '원년' : year + '년'} ${month}월 ${day}일`
			: `${gengou} ${year}, ${day} ${config.months[month - 1]}`;
	}

	function formatBovertDate(
		d: Date,
		config: BovertCalendarConfig,
		korean: boolean = false
	): string {
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
		let dateFormat: string;
		if (dayIndex === config.daysPerSeason - 1) {
			dateFormat = korean ? season.koreanLastDayName : season.lastDayName;
		} else if (days === 365) {
			dateFormat = korean ? config.koreanYearEndDayName : config.yearEndDayName;
		} else {
			const seasonName = korean ? season.koreanName : season.name;
			const dayName = (korean ? config.koreanYearNames : config.yearNames)[dayIndex];
			dateFormat = korean ? `${seasonName} ${dayName}` : `${dayName} ${seasonName}`;
		}

		const names = korean ? config.koreanYearNames : config.yearNames;
		const yearIndex =
			(((year - config.epochYear + config.yearNameOffset) % names.length) + names.length) %
			names.length;
		const yearName = names[yearIndex];

		return korean ? `${yearName} ${dateFormat}` : `${dateFormat} ${yearName}`;
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

<div
	style="display: flex; justify-content: center; align-items: center; height: 100vh; flex-direction: column;"
>
	<div
		style="max-width: 1280px; text-align: center; gap: 32px; display: flex; justify-content: center; align-items: center;"
	>
		<div>
			<div style="font-size: 12px;">자소크력</div>
			{#if imperialCalendar}
				<div style="font-size: 32px;">{formatZasokeseDate(today, imperialCalendar)}</div>
				<div>{formatZasokeseDate(today, imperialCalendar, true)}</div>
			{:else if imperialCalendarError}
				<div role="alert">{imperialCalendarError}</div>
			{:else}
				<div>설정 불러오는 중…</div>
			{/if}
		</div>
		<div>
			<div style="font-size: 12px;">보베르타력</div>
			{#if bovertCalendar}
				<div style="font-size: 32px;">{formatBovertDate(today, bovertCalendar)}</div>
				<div>{formatBovertDate(today, bovertCalendar, true)}</div>
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
		<button type="button" class="clickable" onclick={toPreviousYear}>← 년</button>
		<button type="button" class="clickable" onclick={toPreviousMonth}>← 월</button>
		<button type="button" class="clickable" onclick={toPreviousDay}>← 일</button>
		<button type="button" class="clickable" onclick={toNextDay}>일 →</button>
		<button type="button" class="clickable" onclick={toNextMonth}>월 →</button>
		<button type="button" class="clickable" onclick={toNextYear}>년 →</button>
	</div>
</div>

<style>
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
