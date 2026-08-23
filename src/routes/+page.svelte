<script lang="ts">
	import { onMount } from 'svelte';

	const SKASOJBO = [
		'Сойбо',
		'Шивак',
		'Сиоть',
		'Нязтам',
		'Ракса',
		'Зоста',
		'Жасе',
		'Велек',
		'Добо',
		'Сойвак',
		'Шиоть',
		'Ситам',
		'Нясьа',
		'Ракста',
		'Зосе',
		'Жалек',
		'Вебо',
		'Довак',
		'Соёть',
		'Шитам',
		'Сиса',
		'Нясьта',
		'Раксе',
		'Золек',
		'Жабо',
		'Вевак',
		'Дооть',
		'Сойтам',
		'Шиса',
		'Систа',
		'Нясье',
		'Раклек',
		'Зобо',
		'Жавак',
		'Веоть',
		'Дотам',
		'Сойса',
		'Шиста',
		'Сисе',
		'Нязлек',
		'Ракбо',
		'Зовак',
		'Жаоть',
		'Ветам',
		'Доса',
		'Сойста',
		'Шисе',
		'Силек',
		'Нязбо',
		'Раквак',
		'Зооть',
		'Жатам',
		'Веса',
		'Доста',
		'Сойсе',
		'Шилек',
		'Сибо',
		'Нязвак',
		'Ракоть',
		'Зотам',
		'Жаса',
		'Веста',
		'Досе',
		'Сойлек',
		'Шибо',
		'Сивак',
		'Нязоть',
		'Рактам',
		'Зоса',
		'Жаста',
		'Весе',
		'Долек'
	];
	const SKASOJBO_KOREAN = [
		'소이보',
		'쉬바크',
		'시오트',
		'냐즈탐',
		'라크사',
		'조스타',
		'쟈셰',
		'볠례크',
		'도보',
		'소이바크',
		'쉬오트',
		'시탐',
		'냐사',
		'락스타',
		'조셰',
		'쟐례크',
		'볘보',
		'도바크',
		'소요트',
		'쉬탐',
		'시사',
		'냐스타',
		'라크셰',
		'졸례크',
		'쟈보',
		'볘바크',
		'도오트',
		'소이탐',
		'쉬사',
		'시스타',
		'냐셰',
		'라클례크',
		'조보',
		'쟈바크',
		'볘오트',
		'도탐',
		'소이사',
		'쉬스타',
		'시셰',
		'냐즐례크',
		'라크보',
		'조바크',
		'쟈오트',
		'볘탐',
		'도사',
		'소이스타',
		'쉬셰',
		'실례크',
		'냐즈보',
		'라크바크',
		'조오트',
		'쟈탐',
		'볘사',
		'도스타',
		'소이셰',
		'쉴례크',
		'시보',
		'냐즈바크',
		'라코트',
		'조탐',
		'쟈사',
		'볘스타',
		'도셰',
		'소일례크',
		'쉬보',
		'시바크',
		'냐조트',
		'라크탐',
		'조사',
		'쟈스타',
		'볘셰',
		'돌례크'
	];

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

	const DEFAULT_DATE_OFFSET = 196;
	const MIN_DATE_OFFSET = -730_119; // 0001-01-01
	const MAX_DATE_OFFSET = 2_921_939; // 9999-12-31

	let today = $state(new Date('2000-07-15'));
	let imperialCalendar = $state<ImperialCalendarConfig | null>(null);
	let imperialCalendarError = $state('');

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

	function formatBovertDate(d: Date, korean: boolean = false): string {
		let year = d.getFullYear();

		let days = Math.floor(
			(d.getTime() - new Date(`${d.getFullYear()}-01-01`).getTime()) / (1000 * 60 * 60 * 24) - 104
		);

		if (!(
			(d.getFullYear() % 4 === 0 && d.getFullYear() % 100 !== 0) ||
			d.getFullYear() % 400 === 0
		)) {
			days++;
		}
		if (days < 0) {
			days += 365;
			year--;
		}

		if (((year + 1) % 4 === 0 && (year + 1) % 100 !== 0) || (year + 1) % 400 === 0) {
			if (days >= 261) {
				days++;
			}
		}

		let season;
		if (Math.floor(days / 73) == 0) {
			season = korean ? '화계' : 'Лёг';
		} else if (Math.floor(days / 73) == 1) {
			season = korean ? '서계' : 'Шеж';
		} else if (Math.floor(days / 73) == 2) {
			season = korean ? '양계' : 'Вуча';
		} else if (Math.floor(days / 73) == 3) {
			season = korean ? '설계' : 'Зе';
		} else {
			season = korean ? '용계' : 'Рено';
		}

		let dateFormat;
		if (days % 73 === 72) {
			if (season === 'Лёг') {
				dateFormat = korean ? '크폴잔느료그' : 'Кползаньлёг';
			} else if (season === 'Шеж') {
				dateFormat = korean ? '크폴잔셰즈' : 'Кползаншеж';
			} else if (season === 'Вуча') {
				dateFormat = korean ? '크폴잔부차' : 'Кползанвуча';
			} else if (season === 'Зе') {
				dateFormat = korean ? '크폴잔졔' : 'Кползанзе';
			} else if (season === 'Рено') {
				dateFormat = korean ? '크폴잔느례노' : 'Кползаньрено';
			}
		} else if (days === 365) {
			dateFormat = korean ? '크폴잔큐브' : 'Кползанкюв';
		} else {
			dateFormat = korean
				? `${season} ${SKASOJBO_KOREAN[days % 73]}`
				: `${SKASOJBO[days % 73]} ${season}`;
		}

		const yearIndex = (((year - 2000 + 25) % 72) + 72) % 72;
		const yearName = (korean ? SKASOJBO_KOREAN : SKASOJBO)[yearIndex];

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
			<div style="font-size: 32px;">{formatBovertDate(today)}</div>
			<div>{formatBovertDate(today, true)}</div>
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
