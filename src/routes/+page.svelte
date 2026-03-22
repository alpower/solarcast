<script lang="ts">
	import { onMount } from 'svelte';
	import favicon from '$lib/assets/favicon.svg';

	type PanelString = {
		id: number;
		name: string;
		panelKw: number;
		tilt: number;
		azimuth: number;
		lossPercent: number;
		calibrationFactor: number;
		expanded: boolean;
	};

	type ForecastDay = {
		date: string;
		dayName: string;
		forecastSummary: string;
		avgCloudPercent: number;
		sunnyHours: number;
		generationKwh: number;
		gridImportKwh: number;
		gridExportKwh: number;
		curtailedKwh: number;
		endBatterySocKwh: number;
	};

	type ForecastResult = {
		placeLabel: string;
		latitude: number;
		longitude: number;
		timezone: string;
		days: ForecastDay[];
		totals: {
			generationKwh: number;
			gridImportKwh: number;
			gridExportKwh: number;
			curtailedKwh: number;
		};
	};

	type StoredSettings = {
		locationQuery?: string;
		panelStrings?: PanelString[];
		batteryKwh?: number;
		initialSocPercent?: number;
		roundTripEfficiencyPercent?: number;
		dailyUsageKwh?: number;
		globalIrradianceBiasPercent?: number;
		tempCoefficientPercentPerC?: number;
		noctRiseAt800Wm2?: number;
		sunnyDirectThresholdWm2?: number;
		sunnyCloudMaxPercent?: number;
		exportLimitKw?: number;
		lastResult?: ForecastResult | null;
	};

	const STORAGE_KEY = 'solarcast.settings.v1';
	const ukPostcodePattern = /^([A-Z]{1,2}\d[A-Z\d]?)\s*(\d[A-Z]{2})$/i;
	const compassOptions = [
		{ key: 'N', label: 'North', bearing: 0 },
		{ key: 'NE', label: 'North-East', bearing: 45 },
		{ key: 'E', label: 'East', bearing: 90 },
		{ key: 'SE', label: 'South-East', bearing: 135 },
		{ key: 'S', label: 'South', bearing: 180 },
		{ key: 'SW', label: 'South-West', bearing: 225 },
		{ key: 'W', label: 'West', bearing: 270 },
		{ key: 'NW', label: 'North-West', bearing: 315 }
	] as const;
	type CompassKey = (typeof compassOptions)[number]['key'];

	let locationQuery = $state('Bristol');
	let panelStrings = $state<PanelString[]>([
		{ id: 1, name: 'Main roof', panelKw: 4.0, tilt: 35, azimuth: 0, lossPercent: 14, calibrationFactor: 1, expanded: true }
	]);
	let batteryKwh = $state(6.5);
	let initialSocPercent = $state(50);
	let roundTripEfficiencyPercent = $state(90);
	let dailyUsageKwh = $state(9);
	let globalIrradianceBiasPercent = $state(100);
	let tempCoefficientPercentPerC = $state(-0.4);
	let noctRiseAt800Wm2 = $state(25);
	let sunnyDirectThresholdWm2 = $state(120);
	let sunnyCloudMaxPercent = $state(60);
	let exportLimitKw = $state(3.68);

	let loading = $state(false);
	let errorMessage = $state('');
	let result = $state<ForecastResult | null>(null);
	const chartModel = $derived(result ? buildChartModel(result.days) : null);
	let tooltipVisible = $state(false);
	let tooltipTitle = $state('');
	let tooltipItems = $state<{ label: string; color: string; value: string }[]>([]);
	let tooltipX = $state(0);
	let tooltipY = $state(0);
	let readyToPersist = $state(false);

	const round = (value: number, dp = 2) => Number(value.toFixed(dp));

	function normalizePanelString(input: Partial<PanelString>, id: number): PanelString {
		return {
			id,
			name: String(input.name ?? `String ${id}`).slice(0, 40),
			panelKw: Math.max(0, Number(input.panelKw ?? 0)),
			tilt: Math.max(0, Math.min(90, Number(input.tilt ?? 35))),
			azimuth: Math.max(-180, Math.min(180, Number(input.azimuth ?? 0))),
			lossPercent: Math.max(0, Math.min(80, Number(input.lossPercent ?? 14))),
			calibrationFactor: Math.max(0.5, Math.min(1.5, Number(input.calibrationFactor ?? 1))),
			expanded: typeof input.expanded === 'boolean' ? input.expanded : true
		};
	}

	function getNextStringId() {
		return panelStrings.reduce((max, s) => Math.max(max, s.id), 0) + 1;
	}

	function addPanelString() {
		const id = getNextStringId();
		panelStrings = [
			...panelStrings,
			{
				id,
				name: `String ${id}`,
				panelKw: 1.5,
				tilt: 35,
				azimuth: 0,
				lossPercent: 14,
				calibrationFactor: 1,
				expanded: true
			}
		];
	}

	function removePanelString(id: number) {
		if (panelStrings.length <= 1) {
			return;
		}
		panelStrings = panelStrings.filter((s) => s.id !== id);
	}

	function splitChargeDischargeEfficiency(roundTripEfficiency: number) {
		const r = Math.max(0.5, Math.min(1, roundTripEfficiency / 100));
		const half = Math.sqrt(r);
		return { charge: half, discharge: half };
	}

	function extractDateFromIso(iso: string) {
		return iso.slice(0, 10);
	}

	function dayNameFromDate(date: string) {
		return new Date(`${date}T12:00:00Z`).toLocaleDateString('en-GB', { weekday: 'short' });
	}

	function weatherCodeToSummary(code: number) {
		const map: Record<number, string> = {
			0: 'Clear',
			1: 'Mostly sunny',
			2: 'Partly cloudy',
			3: 'Overcast',
			45: 'Fog',
			48: 'Rime fog',
			51: 'Light drizzle',
			53: 'Drizzle',
			55: 'Heavy drizzle',
			56: 'Freezing drizzle',
			57: 'Heavy freezing drizzle',
			61: 'Light rain',
			63: 'Rain',
			65: 'Heavy rain',
			66: 'Freezing rain',
			67: 'Heavy freezing rain',
			71: 'Light snow',
			73: 'Snow',
			75: 'Heavy snow',
			77: 'Snow grains',
			80: 'Light showers',
			81: 'Showers',
			82: 'Heavy showers',
			85: 'Snow showers',
			86: 'Heavy snow showers',
			95: 'Thunderstorm',
			96: 'Thunderstorm, hail possible',
			99: 'Thunderstorm, heavy hail possible'
		};

		return map[code] ?? 'Mixed conditions';
	}

	function safeArray<T>(value: unknown): T[] {
		return Array.isArray(value) ? (value as T[]) : [];
	}

	function formatDateForRange(date: string) {
		return new Date(`${date}T12:00:00Z`).toLocaleDateString('en-GB', { day: 'numeric', month: 'short' });
	}

	function forecastDateRange(days: ForecastDay[]) {
		if (!days.length) return '';
		const start = formatDateForRange(days[0].date);
		const end = formatDateForRange(days[days.length - 1].date);
		return `${start} to ${end}`;
	}

	function weatherIconFromSummary(summary: string) {
		const s = summary.toLowerCase();
		if (s.includes('thunder')) return '⛈️';
		if (s.includes('snow')) return '❄️';
		if (s.includes('freezing')) return '🥶';
		if (s.includes('drizzle')) return '🌦️';
		if (s.includes('rain') || s.includes('showers')) return '🌧️';
		if (s.includes('fog')) return '🌫️';
		if (s.includes('overcast')) return '☁️';
		if (s.includes('partly')) return '⛅';
		if (s.includes('mostly sunny') || s.includes('clear')) return '☀️';
		return '🌤️';
	}

	function showTooltip(event: MouseEvent | FocusEvent, title: string, items: { label: string; color: string; value: string }[]) {
		const target = event.currentTarget as Element | null;
		if (!target) return;
		const container = target?.closest('.chart-wrap') as HTMLElement | null;
		if (!container) return;

		const containerRect = container.getBoundingClientRect();
		const targetRect = target.getBoundingClientRect();
		const clientX = 'clientX' in event ? event.clientX : targetRect.left + targetRect.width / 2;
		const clientY = 'clientY' in event ? event.clientY : targetRect.top;
		tooltipTitle = title;
		tooltipItems = items;
		tooltipX = clientX - containerRect.left + 10;
		tooltipY = clientY - containerRect.top - 12;
		tooltipVisible = true;
	}

	function hideTooltip() {
		tooltipVisible = false;
		tooltipTitle = '';
		tooltipItems = [];
	}

	function buildChartModel(days: ForecastDay[]) {
		const chart = {
			width: 860,
			height: 320,
			marginTop: 34,
			marginRight: 16,
			marginBottom: 92,
			marginLeft: 44
		};
		const innerWidth = chart.width - chart.marginLeft - chart.marginRight;
		const innerHeight = chart.height - chart.marginTop - chart.marginBottom;
		const dayCount = Math.max(1, days.length);
		const groupWidth = innerWidth / dayCount;
		const barWidth = Math.max(6, Math.min(18, groupWidth * 0.16));
		const seriesBase = [
			{ key: 'generationKwh', color: '#16a34a', label: 'Generation' },
			{ key: 'gridImportKwh', color: '#0ea5e9', label: 'Import' },
			{ key: 'gridExportKwh', color: '#f59e0b', label: 'Export' },
			{ key: 'curtailedKwh', color: '#ef4444', label: 'Curtailed' }
		] as const;
		const series = seriesBase.map((s) => ({
			...s,
			hasData: days.some((d) => d[s.key] > 0)
		}));

		const maxEnergyRaw = Math.max(
			0.1,
			...days.flatMap((d) => [d.generationKwh, d.gridImportKwh, d.gridExportKwh, d.curtailedKwh])
		);
		const maxEnergy = Math.max(1, Math.ceil(maxEnergyRaw / 5) * 5);
		const energyTicks = [0, 0.25, 0.5, 0.75, 1].map((r) => maxEnergy * r);
		const cloudPoints = days.map((d, i) => {
			const x = chart.marginLeft + groupWidth * i + groupWidth / 2;
			const y = chart.marginTop + innerHeight * (1 - d.avgCloudPercent / 100);
			return { i, x, y, cloud: d.avgCloudPercent };
		});
		const cloudPath = days
			.map((d, i) => {
				const x = chart.marginLeft + groupWidth * i + groupWidth / 2;
				const y = chart.marginTop + innerHeight * (1 - d.avgCloudPercent / 100);
				return `${i === 0 ? 'M' : 'L'} ${x.toFixed(1)} ${y.toFixed(1)}`;
			})
			.join(' ');

		return { chart, innerWidth, innerHeight, groupWidth, barWidth, series, maxEnergy, energyTicks, cloudPoints, cloudPath };
	}

	function normalizeAzimuth(value: number) {
		if (value > 180) return value - 360;
		if (value <= -180) return value + 360;
		return value;
	}

	function azimuthFromCompassKey(key: string) {
		const option = compassOptions.find((d) => d.key === key) ?? compassOptions[4];
		return normalizeAzimuth(option.bearing - 180);
	}

	function compassKeyFromAzimuth(azimuth: number) {
		const normalized = normalizeAzimuth(azimuth);
		let best: CompassKey = 'S';
		let bestDistance = Number.POSITIVE_INFINITY;

		for (const option of compassOptions) {
			const target = azimuthFromCompassKey(option.key);
			const raw = Math.abs(normalized - target);
			const wrapped = Math.min(raw, 360 - raw);
			if (wrapped < bestDistance) {
				bestDistance = wrapped;
				best = option.key;
			}
		}

		return best;
	}

	function compassLabelFromAzimuth(azimuth: number) {
		const key = compassKeyFromAzimuth(azimuth);
		return compassOptions.find((option) => option.key === key)?.label ?? 'South';
	}

	onMount(() => {
		try {
			const raw = localStorage.getItem(STORAGE_KEY);
			if (!raw) {
				readyToPersist = true;
				return;
			}

			const parsed = JSON.parse(raw) as StoredSettings;
			if (typeof parsed.locationQuery === 'string') {
				locationQuery = parsed.locationQuery;
			}

			if (Array.isArray(parsed.panelStrings) && parsed.panelStrings.length > 0) {
				panelStrings = parsed.panelStrings.map((item, index) => normalizePanelString(item, index + 1));
			}

			if (typeof parsed.batteryKwh === 'number') batteryKwh = Math.max(0, parsed.batteryKwh);
			if (typeof parsed.initialSocPercent === 'number') {
				initialSocPercent = Math.max(0, Math.min(100, parsed.initialSocPercent));
			}
			if (typeof parsed.roundTripEfficiencyPercent === 'number') {
				roundTripEfficiencyPercent = Math.max(50, Math.min(100, parsed.roundTripEfficiencyPercent));
			}
			if (typeof parsed.dailyUsageKwh === 'number') dailyUsageKwh = Math.max(0, parsed.dailyUsageKwh);
			if (typeof parsed.globalIrradianceBiasPercent === 'number') {
				globalIrradianceBiasPercent = Math.max(70, Math.min(140, parsed.globalIrradianceBiasPercent));
			}
			if (typeof parsed.tempCoefficientPercentPerC === 'number') {
				tempCoefficientPercentPerC = Math.max(-0.7, Math.min(-0.2, parsed.tempCoefficientPercentPerC));
			}
			if (typeof parsed.noctRiseAt800Wm2 === 'number') {
				noctRiseAt800Wm2 = Math.max(10, Math.min(40, parsed.noctRiseAt800Wm2));
			}
			if (typeof parsed.sunnyDirectThresholdWm2 === 'number') {
				sunnyDirectThresholdWm2 = Math.max(50, Math.min(400, parsed.sunnyDirectThresholdWm2));
			}
			if (typeof parsed.sunnyCloudMaxPercent === 'number') {
				sunnyCloudMaxPercent = Math.max(10, Math.min(90, parsed.sunnyCloudMaxPercent));
			}
			if (typeof parsed.exportLimitKw === 'number') {
				exportLimitKw = Math.max(0, Math.min(20, parsed.exportLimitKw));
			}
			if (parsed.lastResult && Array.isArray(parsed.lastResult.days) && parsed.lastResult.days.length > 0) {
				result = parsed.lastResult;
			}
		} catch {
			localStorage.removeItem(STORAGE_KEY);
		} finally {
			readyToPersist = true;
		}
	});

	$effect(() => {
		if (!readyToPersist) return;
		if (typeof localStorage === 'undefined') return;

		const payload: StoredSettings = {
			locationQuery,
			panelStrings,
			batteryKwh,
			initialSocPercent,
			roundTripEfficiencyPercent,
			dailyUsageKwh,
			globalIrradianceBiasPercent,
			tempCoefficientPercentPerC,
			noctRiseAt800Wm2,
			sunnyDirectThresholdWm2,
			sunnyCloudMaxPercent,
			exportLimitKw,
			lastResult: result
		};

		localStorage.setItem(STORAGE_KEY, JSON.stringify(payload));
	});

	async function buildForecast() {
		loading = true;
		errorMessage = '';
		result = null;
		panelStrings = panelStrings.map((panel) => ({ ...panel, expanded: false }));

		try {
			const activeStrings = panelStrings
				.map((s) => ({
					...s,
					panelKw: Math.max(0, Number(s.panelKw)),
					tilt: Math.max(0, Math.min(90, Number(s.tilt))),
					azimuth: Math.max(-180, Math.min(180, Number(s.azimuth))),
					lossPercent: Math.max(0, Math.min(80, Number(s.lossPercent))),
					calibrationFactor: Math.max(0.5, Math.min(1.5, Number(s.calibrationFactor)))
				}))
				.filter((s) => s.panelKw > 0);

			if (!activeStrings.length) {
				throw new Error('Add at least one panel string with kWp > 0.');
			}

			const query = locationQuery.trim();
			const compactPostcode = query.replace(/\s+/g, '').toUpperCase();
			const isPostcode = ukPostcodePattern.test(query);

			let lat = Number.NaN;
			let lon = Number.NaN;
			let placeLabel = '';

			if (isPostcode) {
				const postcodesRes = await fetch(`https://api.postcodes.io/postcodes/${encodeURIComponent(compactPostcode)}`);
				const postcodesJson = await postcodesRes.json();

				if (postcodesRes.ok && postcodesJson?.status === 200 && postcodesJson?.result) {
					lat = Number(postcodesJson.result.latitude);
					lon = Number(postcodesJson.result.longitude);
					const postcode = String(postcodesJson.result.postcode ?? compactPostcode);
					const district = String(postcodesJson.result.admin_district ?? '');
					placeLabel = district ? `${postcode}, ${district}` : postcode;
				}
			}

			if (!Number.isFinite(lat) || !Number.isFinite(lon)) {
				const geocodeUrl = new URL('https://geocoding-api.open-meteo.com/v1/search');
				geocodeUrl.searchParams.set('name', query);
				geocodeUrl.searchParams.set('count', '1');
				geocodeUrl.searchParams.set('language', 'en');
				geocodeUrl.searchParams.set('format', 'json');
				geocodeUrl.searchParams.set('countryCode', 'GB');

				const geoRes = await fetch(geocodeUrl);
				if (!geoRes.ok) {
					throw new Error(`Geocoding failed (${geoRes.status})`);
				}

				const geoJson = await geoRes.json();
				const places = safeArray<Record<string, unknown>>(geoJson?.results);
				if (!places.length) {
					throw new Error('No UK location match found. Try a city, town, or full postcode.');
				}

				const place = places[0];
				lat = Number(place.latitude);
				lon = Number(place.longitude);
				const placeName = String(place.name ?? 'Unknown');
				const admin1 = String(place.admin1 ?? '');
				placeLabel = admin1 ? `${placeName}, ${admin1}` : placeName;
			}

			if (!Number.isFinite(lat) || !Number.isFinite(lon)) {
				throw new Error('Location coordinates were invalid.');
			}

			const dailyWeatherUrl = new URL('https://api.open-meteo.com/v1/forecast');
			dailyWeatherUrl.searchParams.set('latitude', String(lat));
			dailyWeatherUrl.searchParams.set('longitude', String(lon));
			dailyWeatherUrl.searchParams.set('daily', 'weather_code');
			dailyWeatherUrl.searchParams.set('forecast_days', '7');
			dailyWeatherUrl.searchParams.set('timezone', 'auto');

			const dailyRes = await fetch(dailyWeatherUrl);
			if (!dailyRes.ok) {
				throw new Error(`Daily forecast request failed (${dailyRes.status})`);
			}

			const dailyJson = await dailyRes.json();
			const dailyDates = safeArray<string>(dailyJson?.daily?.time);
			const dailyCodes = safeArray<number>(dailyJson?.daily?.weather_code);
			const forecastByDate = new Map<string, string>();
			for (let i = 0; i < Math.min(dailyDates.length, dailyCodes.length); i += 1) {
				forecastByDate.set(dailyDates[i], weatherCodeToSummary(Number(dailyCodes[i])));
			}

			const hourlyWeatherUrl = new URL('https://api.open-meteo.com/v1/forecast');
			hourlyWeatherUrl.searchParams.set('latitude', String(lat));
			hourlyWeatherUrl.searchParams.set('longitude', String(lon));
			hourlyWeatherUrl.searchParams.set('hourly', 'temperature_2m,cloud_cover,direct_radiation');
			hourlyWeatherUrl.searchParams.set('forecast_days', '7');
			hourlyWeatherUrl.searchParams.set('timezone', 'auto');

			const hourlyWeatherRes = await fetch(hourlyWeatherUrl);
			if (!hourlyWeatherRes.ok) {
				throw new Error(`Hourly weather request failed (${hourlyWeatherRes.status})`);
			}

			const hourlyWeatherJson = await hourlyWeatherRes.json();
			const weatherTimes = safeArray<string>(hourlyWeatherJson?.hourly?.time);
			const temperatures = safeArray<number>(hourlyWeatherJson?.hourly?.temperature_2m);
			const cloudCover = safeArray<number>(hourlyWeatherJson?.hourly?.cloud_cover);
			const directRadiation = safeArray<number>(hourlyWeatherJson?.hourly?.direct_radiation);

			const forecasts = await Promise.all(
				activeStrings.map(async (s) => {
					const forecastUrl = new URL('https://api.open-meteo.com/v1/forecast');
					forecastUrl.searchParams.set('latitude', String(lat));
					forecastUrl.searchParams.set('longitude', String(lon));
					forecastUrl.searchParams.set('hourly', 'global_tilted_irradiance');
					forecastUrl.searchParams.set('tilt', String(s.tilt));
					forecastUrl.searchParams.set('azimuth', String(s.azimuth));
					forecastUrl.searchParams.set('forecast_days', '7');
					forecastUrl.searchParams.set('timezone', 'auto');

					const wxRes = await fetch(forecastUrl);
					if (!wxRes.ok) {
						throw new Error(`Forecast request failed (${wxRes.status})`);
					}

					const wxJson = await wxRes.json();
					return {
						times: safeArray<string>(wxJson?.hourly?.time),
						gti: safeArray<number>(wxJson?.hourly?.global_tilted_irradiance),
						timezone: String(wxJson?.timezone ?? 'UTC'),
						panelKw: s.panelKw,
						systemDerate: 1 - s.lossPercent / 100,
						calibrationFactor: s.calibrationFactor
					};
				})
			);

			const primary = forecasts[0];
			if (!primary.times.length || !primary.gti.length || primary.times.length !== primary.gti.length) {
				throw new Error('Forecast response was missing hourly solar data.');
			}

			for (const f of forecasts) {
				if (f.times.length !== primary.times.length || f.gti.length !== primary.gti.length) {
					throw new Error('Forecast data mismatch between panel strings.');
				}
			}
			if (
				weatherTimes.length !== primary.times.length ||
				temperatures.length !== primary.times.length ||
				cloudCover.length !== primary.times.length ||
				directRadiation.length !== primary.times.length
			) {
				throw new Error('Hourly weather alignment error. Please try again.');
			}

			const batteryCapacity = Math.max(0, batteryKwh);
			const usagePerHour = Math.max(0, dailyUsageKwh) / 24;
			const eff = splitChargeDischargeEfficiency(roundTripEfficiencyPercent);
			const biasFactor = Math.max(0.7, Math.min(1.4, globalIrradianceBiasPercent / 100));
			const tempCoeff = Math.max(-0.007, Math.min(-0.002, tempCoefficientPercentPerC / 100));
			const noctRiseFactor = Math.max(10, Math.min(40, noctRiseAt800Wm2));
			const sunnyDirectThreshold = Math.max(50, Math.min(400, sunnyDirectThresholdWm2));
			const sunnyCloudMax = Math.max(10, Math.min(90, sunnyCloudMaxPercent));
			const exportPowerLimitKw = Math.max(0, Math.min(20, exportLimitKw));
			let batterySoc = batteryCapacity * Math.max(0, Math.min(100, initialSocPercent)) / 100;
			const byDate = new Map<string, ForecastDay>();
			const dayHourCounts = new Map<string, number>();
			const dayCloudSums = new Map<string, number>();

			for (let i = 0; i < primary.times.length; i += 1) {
				const date = extractDateFromIso(primary.times[i]);
				const generationKwh = forecasts.reduce((sum, f) => {
					const irradianceWm2 = Number(f.gti[i]) || 0;
					const ambientTemp = Number(temperatures[i]) || 10;
					const moduleTemp = ambientTemp + (irradianceWm2 / 800) * noctRiseFactor;
					const temperatureFactor = Math.max(0.75, Math.min(1.1, 1 + tempCoeff * (moduleTemp - 25)));
					return (
						sum +
						Math.max(
							0,
							f.panelKw *
								(irradianceWm2 / 1000) *
								f.systemDerate *
								temperatureFactor *
								f.calibrationFactor *
								biasFactor
						)
					);
				}, 0);
				const cloud = Math.max(0, Math.min(100, Number(cloudCover[i]) || 0));
				const direct = Math.max(0, Number(directRadiation[i]) || 0);
				const isSunnyHour = direct >= sunnyDirectThreshold && cloud <= sunnyCloudMax;

				const netKwh = generationKwh - usagePerHour;
				let gridImport = 0;
				let gridExport = 0;
				let curtailed = 0;

				if (netKwh >= 0) {
					const storable = (batteryCapacity - batterySoc) / eff.charge;
					const chargeIn = Math.max(0, Math.min(netKwh, storable));
					batterySoc += chargeIn * eff.charge;
					const exportBeforeLimit = netKwh - chargeIn;
					// Model timestep is 1 hour, so kW limit maps to max hourly exported energy.
					gridExport = Math.min(exportBeforeLimit, exportPowerLimitKw);
					curtailed = Math.max(0, exportBeforeLimit - gridExport);
				} else {
					const deficit = -netKwh;
					const maxDeliverable = batterySoc * eff.discharge;
					const discharged = Math.min(deficit, maxDeliverable);
					batterySoc -= discharged / eff.discharge;
					gridImport = deficit - discharged;
				}

				const day = byDate.get(date) ?? {
					date,
					dayName: dayNameFromDate(date),
					forecastSummary: forecastByDate.get(date) ?? 'Forecast unavailable',
					avgCloudPercent: 0,
					sunnyHours: 0,
					generationKwh: 0,
					gridImportKwh: 0,
					gridExportKwh: 0,
					curtailedKwh: 0,
					endBatterySocKwh: batterySoc
				};

				day.generationKwh += generationKwh;
				day.gridImportKwh += gridImport;
				day.gridExportKwh += gridExport;
				day.curtailedKwh += curtailed;
				day.endBatterySocKwh = batterySoc;
				day.sunnyHours += isSunnyHour ? 1 : 0;
				byDate.set(date, day);

				dayHourCounts.set(date, (dayHourCounts.get(date) ?? 0) + 1);
				dayCloudSums.set(date, (dayCloudSums.get(date) ?? 0) + cloud);
			}

			const days = [...byDate.values()]
				.sort((a, b) => a.date.localeCompare(b.date))
				.slice(0, 7)
				.map((d) => ({
					...d,
					avgCloudPercent: round((dayCloudSums.get(d.date) ?? 0) / Math.max(1, dayHourCounts.get(d.date) ?? 1), 0),
					sunnyHours: round(d.sunnyHours, 0),
					generationKwh: round(d.generationKwh),
					gridImportKwh: round(d.gridImportKwh),
					gridExportKwh: round(d.gridExportKwh),
					curtailedKwh: round(d.curtailedKwh),
					endBatterySocKwh: round(d.endBatterySocKwh)
				}));

			result = {
				placeLabel,
				latitude: round(lat, 4),
				longitude: round(lon, 4),
				timezone: primary.timezone,
				days,
				totals: {
					generationKwh: round(days.reduce((sum, d) => sum + d.generationKwh, 0)),
					gridImportKwh: round(days.reduce((sum, d) => sum + d.gridImportKwh, 0)),
					gridExportKwh: round(days.reduce((sum, d) => sum + d.gridExportKwh, 0)),
					curtailedKwh: round(days.reduce((sum, d) => sum + d.curtailedKwh, 0))
				}
			};
		} catch (error) {
			errorMessage = error instanceof Error ? error.message : 'Unexpected error while building forecast.';
		} finally {
			loading = false;
		}
	}
</script>

<svelte:head>
	<title>Solarcast</title>
	<meta
		name="description"
		content="UK week-ahead solar generation estimate using Open-Meteo weather data and your system setup."
	/>
</svelte:head>

<main>
	<div class="app-layout">
		<div class="left-col">
			<section class="card form-card">
				<h1 class="brand-title"><img src={favicon} alt="" class="brand-icon" />Solarcast</h1>
				<p class="subtitle">7-day UK solar estimate powered by Open-Meteo</p>

				<form
					onsubmit={(event) => {
						event.preventDefault();
						void buildForecast();
					}}
				>
					<label>
						Location (UK)
						<span class="helper-text">Use UK town or postcode</span>
						<input data-1p-ignore data-lpignore="true" bind:value={locationQuery} placeholder="e.g. Bristol or SW1A 1AA" required />
					</label>

					<div class="panel-section">
						<div class="section-head">
							<h3>Panel strings</h3>
							<button class="ghost" type="button" onclick={addPanelString}>Add string</button>
						</div>

						{#each panelStrings as panel (panel.id)}
							<details
								class="panel-item"
								open={panel.expanded}
								ontoggle={(event) => {
									panel.expanded = (event.currentTarget as HTMLDetailsElement).open;
								}}
							>
								<summary>
									<div class="panel-summary-row">
										<div class="panel-summary-copy">
											<span class="panel-title">{panel.name || `String ${panel.id}`}</span>
											<span class="panel-meta">
												{Number(panel.panelKw || 0).toFixed(1)} kWp · {compassLabelFromAzimuth(panel.azimuth)} ·
												tilt {Math.round(panel.tilt || 0)}° · loss {Math.round(panel.lossPercent || 0)}% · cal{' '}
												{Number(panel.calibrationFactor || 1).toFixed(2)}
											</span>
										</div>
										<span class="panel-toggle" aria-hidden="true">
											<span class="panel-toggle-text">{panel.expanded ? 'Collapse' : 'Expand'}</span>
											<span class="panel-chevron">▾</span>
										</span>
									</div>
								</summary>
								<div class="panel-fields">
									<label>
										Name
										<input data-1p-ignore data-lpignore="true" bind:value={panel.name} placeholder={`String ${panel.id}`} />
									</label>
									<label>
										kWp
										<input data-1p-ignore data-lpignore="true" bind:value={panel.panelKw} type="number" min="0" step="0.1" />
									</label>
									<label>
										Tilt
										<input data-1p-ignore data-lpignore="true" bind:value={panel.tilt} type="number" min="0" max="90" step="1" />
									</label>
									<label>
										Direction
										<select
											value={compassKeyFromAzimuth(panel.azimuth)}
											onchange={(event) => {
												const value = (event.currentTarget as HTMLSelectElement).value;
												panel.azimuth = azimuthFromCompassKey(value);
											}}
										>
											{#each compassOptions as option}
												<option value={option.key}>{option.label}</option>
											{/each}
										</select>
									</label>
									<label>
										Losses %
										<input data-1p-ignore data-lpignore="true" bind:value={panel.lossPercent} type="number" min="0" max="80" step="1" />
									</label>
									<label>
										Cal
										<input data-1p-ignore data-lpignore="true" bind:value={panel.calibrationFactor} type="number" min="0.5" max="1.5" step="0.01" />
									</label>
								</div>
								<div class="panel-actions">
									<button
										type="button"
										class="remove"
										onclick={() => removePanelString(panel.id)}
										disabled={panelStrings.length <= 1}
									>
										Remove string
									</button>
								</div>
							</details>
						{/each}
					</div>

					<div class="grid">
						<label>
							Battery (kWh)
							<input data-1p-ignore data-lpignore="true" bind:value={batteryKwh} type="number" min="0" step="0.1" />
						</label>

						<label>
							Start SoC (%)
							<input data-1p-ignore data-lpignore="true" bind:value={initialSocPercent} type="number" min="0" max="100" step="1" />
						</label>

						<label>
							Round-trip efficiency (%)
							<input data-1p-ignore data-lpignore="true" bind:value={roundTripEfficiencyPercent} type="number" min="50" max="100" step="1" />
						</label>

						<label>
							Daily usage (kWh)
							<input data-1p-ignore data-lpignore="true" bind:value={dailyUsageKwh} type="number" min="0" step="0.1" />
						</label>
					</div>

					<details class="advanced">
						<summary>Advanced forecast tuning</summary>
						<div class="grid">
							<label>
								Forecast bias (%)
								<span class="helper-text">Overall output scaling. Raise if forecasts are consistently low.</span>
								<input data-1p-ignore data-lpignore="true" bind:value={globalIrradianceBiasPercent} type="number" min="70" max="140" step="1" />
							</label>
							<label>
								Temp coeff (%/C)
								<span class="helper-text">Panel power change per deg C above 25C (usually negative).</span>
								<input data-1p-ignore data-lpignore="true" bind:value={tempCoefficientPercentPerC} type="number" min="-0.7" max="-0.2" step="0.01" />
							</label>
							<label>
								NOCT rise @800W/m2 (C)
								<span class="helper-text">Estimated panel temperature rise above ambient at strong sun.</span>
								<input data-1p-ignore data-lpignore="true" bind:value={noctRiseAt800Wm2} type="number" min="10" max="40" step="1" />
							</label>
							<label>
								Sunny direct threshold
								<span class="helper-text">Hourly direct radiation needed to count as a sunny hour.</span>
								<input data-1p-ignore data-lpignore="true" bind:value={sunnyDirectThresholdWm2} type="number" min="50" max="400" step="10" />
							</label>
							<label>
								Sunny cloud max (%)
								<span class="helper-text">Maximum cloud cover allowed for a sunny-hour count.</span>
								<input data-1p-ignore data-lpignore="true" bind:value={sunnyCloudMaxPercent} type="number" min="10" max="90" step="1" />
							</label>
							<label>
								Export limit (kW, instant)
								<span class="helper-text">Grid export power cap from your inverter/export settings.</span>
								<input data-1p-ignore data-lpignore="true" bind:value={exportLimitKw} type="number" min="0" max="20" step="0.01" />
							</label>
						</div>
					</details>

					<button disabled={loading}>
						{#if loading}
							Calculating...
						{:else if result}
							Rebuild forecast
						{:else}
							Build forecast
						{/if}
					</button>
				</form>
			</section>

			{#if errorMessage}
				<p class="error">{errorMessage}</p>
			{/if}
		</div>

		<div class="right-col">
			{#if result}
				<section class="card results">
					<div class="results-head">
						<div>
							<h2>{result.placeLabel}</h2>
							<p class="meta">
								<strong>{forecastDateRange(result.days)}</strong> · {result.latitude}, {result.longitude} (
								{result.timezone})
							</p>
						</div>
						<button type="button" class="ghost" onclick={() => void buildForecast()} disabled={loading}>
							Rebuild forecast
						</button>
					</div>

					<div class="totals">
						<div>
							<span>Week generation</span>
							<strong>{result.totals.generationKwh} kWh</strong>
						</div>
						<div>
							<span>Grid import</span>
							<strong>{result.totals.gridImportKwh} kWh</strong>
						</div>
						<div>
							<span>Grid export</span>
							<strong>{result.totals.gridExportKwh} kWh</strong>
						</div>
						<div>
							<span>Curtailed</span>
							<strong>{result.totals.curtailedKwh} kWh</strong>
						</div>
					</div>

					<div class="chart-wrap">
						<div class="legend">
							{#if chartModel}
								{#each chartModel.series as s}
									<span class={`chip ${s.hasData ? '' : 'chip-disabled'}`}>
										<span class="swatch" style={`background:${s.color}`}></span>
										{s.label}
									</span>
								{/each}
							{/if}
							<span class="chip">
								<span class="swatch line-swatch"></span>
								Cloud %
							</span>
						</div>
						{#if chartModel}
							<svg viewBox={`0 0 ${chartModel.chart.width} ${chartModel.chart.height}`} aria-label="Weekly weather and energy chart">
								<defs>
									<linearGradient id="chart-bg" x1="0" y1="0" x2="0" y2="1">
										<stop offset="0%" stop-color="#f8fbff" />
										<stop offset="100%" stop-color="#f1f5f9" />
									</linearGradient>
								</defs>
								<rect
									x={chartModel.chart.marginLeft - 8}
									y={chartModel.chart.marginTop - 8}
									width={chartModel.innerWidth + 16}
									height={chartModel.innerHeight + 16}
									rx="10"
									fill="url(#chart-bg)"
								/>
								{#each chartModel.energyTicks as tick}
									{@const yTick = chartModel.chart.marginTop + chartModel.innerHeight * (1 - tick / chartModel.maxEnergy)}
									<line
										x1={chartModel.chart.marginLeft}
										y1={yTick}
										x2={chartModel.chart.marginLeft + chartModel.innerWidth}
										y2={yTick}
										stroke="#d9e2ef"
										stroke-dasharray="3 4"
									/>
									<text x={chartModel.chart.marginLeft - 8} y={yTick + 3} text-anchor="end" class="axis">
										{tick}
									</text>
								{/each}

								<line
									x1={chartModel.chart.marginLeft}
									y1={chartModel.chart.marginTop + chartModel.innerHeight}
									x2={chartModel.chart.marginLeft + chartModel.innerWidth}
									y2={chartModel.chart.marginTop + chartModel.innerHeight}
									stroke="#94a3b8"
								/>
								<line
									x1={chartModel.chart.marginLeft}
									y1={chartModel.chart.marginTop}
									x2={chartModel.chart.marginLeft}
									y2={chartModel.chart.marginTop + chartModel.innerHeight}
									stroke="#94a3b8"
								/>

								{#each result.days as day, i}
									{@const groupX = chartModel.chart.marginLeft + chartModel.groupWidth * i}
									<rect
										x={groupX}
										y={chartModel.chart.marginTop}
										width={chartModel.groupWidth}
										height={chartModel.innerHeight}
										fill="transparent"
										role="img"
										aria-label={`${day.dayName} ${day.date} grouped data`}
										onmousemove={(event) =>
											showTooltip(event, `${day.dayName} ${day.date}`, [
												{ label: 'Generation', color: chartModel.series[0].color, value: `${day.generationKwh} kWh` },
												{ label: 'Import', color: chartModel.series[1].color, value: `${day.gridImportKwh} kWh` },
												{ label: 'Export', color: chartModel.series[2].color, value: `${day.gridExportKwh} kWh` },
												{ label: 'Curtailed', color: chartModel.series[3].color, value: `${day.curtailedKwh} kWh` },
												{ label: 'Cloud', color: '#7c3aed', value: `${day.avgCloudPercent}%` }
											])}
										onmouseleave={hideTooltip}
									/>
									{#each chartModel.series as s, si}
										{@const value = day[s.key]}
										{@const h = (value / chartModel.maxEnergy) * chartModel.innerHeight}
										{@const x = chartModel.chart.marginLeft + chartModel.groupWidth * i + si * chartModel.barWidth + 6}
										{@const y = chartModel.chart.marginTop + chartModel.innerHeight - h}
										<rect x={x} y={y} width={chartModel.barWidth - 2} height={h} fill={s.color} rx="2" />
									{/each}
									{@const dayX = chartModel.chart.marginLeft + chartModel.groupWidth * i + chartModel.groupWidth / 2}
									<text
										x={dayX}
										y={chartModel.chart.marginTop + chartModel.innerHeight + 24}
										text-anchor="middle"
										class="axis x-axis-label"
									>
										{day.dayName}
									</text>
									<text x={dayX} y={chartModel.chart.marginTop + chartModel.innerHeight + 64} text-anchor="middle" class="weather-icon">
										{weatherIconFromSummary(day.forecastSummary)}
									</text>
								{/each}

								<path d={chartModel.cloudPath} fill="none" stroke="#7c3aed" stroke-width="2.5" />
								{#each chartModel.cloudPoints as p}
									<circle cx={p.x} cy={p.y} r="4" fill="#7c3aed" />
								{/each}

								<text x={2} y={chartModel.chart.marginTop - 14} class="axis y-axis-label">Energy (kWh)</text>
								<text
									x={chartModel.chart.marginLeft + chartModel.innerWidth - 6}
									y={chartModel.chart.marginTop + 12}
									class="axis"
									text-anchor="end"
								>
									Cloud %
								</text>
							</svg>
							{#if tooltipVisible}
								<div class="chart-tooltip" style={`left:${tooltipX}px;top:${tooltipY}px;`}>
									<div class="tooltip-title">{tooltipTitle}</div>
									{#each tooltipItems as item}
										<div class="tooltip-row">
											<span class="swatch" style={`background:${item.color}`}></span>
											<span>{item.label}</span>
											<strong>{item.value}</strong>
										</div>
									{/each}
								</div>
							{/if}
						{/if}
					</div>

					<table>
						<thead>
							<tr>
								<th>Date</th>
								<th>Day</th>
								<th>Forecast</th>
								<th>Cloud</th>
								<th>Sunny hrs</th>
								<th>Generation</th>
								<th>Import</th>
								<th>Export</th>
								<th>Curtailed</th>
								<th>End battery SoC</th>
							</tr>
						</thead>
						<tbody>
							{#each result.days as day}
								<tr>
									<td>{day.date}</td>
									<td>{day.dayName}</td>
									<td>{day.forecastSummary}</td>
									<td>{day.avgCloudPercent}%</td>
									<td>{day.sunnyHours}</td>
									<td>{day.generationKwh} kWh</td>
									<td>{day.gridImportKwh} kWh</td>
									<td>{day.gridExportKwh} kWh</td>
									<td>{day.curtailedKwh} kWh</td>
									<td>{day.endBatterySocKwh} kWh</td>
								</tr>
							{/each}
						</tbody>
					</table>

					<p class="note">
						Azimuth means panel compass direction. This app uses Open-Meteo convention: South = 0, East = -90,
						West = 90, North = +/-180.
					</p>
					<p class="note">
						Generation now includes a panel temperature derate model and optional per-string calibration factor
						(1.00 = baseline).
					</p>
					<p class="note">
						If your output is consistently higher than forecast, increase Forecast bias above 100% or raise a
						string's Cal value above 1.00.
					</p>
					<p class="note">
						Export limit is an instantaneous kW cap. In this hourly model, we apply it to each hour's average export.
						Extra energy beyond battery + export cap is shown as curtailed.
					</p>
					<p class="note">
						Model notes: this is a simplified estimate using forecast irradiance per panel string, then battery and
						fixed demand simulation.
					</p>
				</section>
			{:else}
				<section class="card results placeholder">
					<div class="empty-state-icon" aria-hidden="true">🌤️</div>
					<h2 class="empty-state-title">Forecast Awaits</h2>
					<p class="meta empty-state-copy">Set your inputs on the left and click Build forecast.</p>
				</section>
			{/if}
		</div>
	</div>
</main>

<style>
	:global(body) {
		margin: 0;
		font-family: 'Avenir Next', Avenir, 'Segoe UI', sans-serif;
		background: radial-gradient(circle at top, #dbeafe 0%, #f8fafc 50%, #eef2ff 100%);
		color: #0f172a;
	}

	main {
		max-width: 1400px;
		margin: 0 auto;
		padding: 2rem 1rem 3rem;
	}

	.app-layout {
		display: grid;
		grid-template-columns: 1fr;
		gap: 1rem;
	}

	.left-col,
	.right-col {
		min-width: 0;
	}

	.right-col {
		display: flex;
	}

	.card {
		background: rgba(255, 255, 255, 0.9);
		backdrop-filter: blur(4px);
		border-radius: 1rem;
		padding: 1.25rem;
		box-shadow: 0 12px 30px rgba(15, 23, 42, 0.08);
	}

	h1 {
		margin: 0;
		font-size: 2rem;
	}

	.brand-title {
		display: flex;
		align-items: center;
		gap: 0.55rem;
	}

	.brand-icon {
		width: 1.6rem;
		height: 1.6rem;
		flex-shrink: 0;
	}

	h3 {
		margin: 0;
		font-size: 1rem;
	}

	.subtitle {
		margin: 0.25rem 0 1.25rem;
		color: #475569;
	}

	form {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	label {
		display: flex;
		flex-direction: column;
		gap: 0.35rem;
		font-size: 0.92rem;
		font-weight: 600;
	}

	.helper-text {
		font-size: 0.78rem;
		font-weight: 500;
		color: #64748b;
	}

	input {
		font: inherit;
		padding: 0.55rem 0.7rem;
		border: 1px solid #cbd5e1;
		border-radius: 0.55rem;
		background: #fff;
	}

	select {
		font: inherit;
		padding: 0.55rem 0.7rem;
		border: 1px solid #cbd5e1;
		border-radius: 0.55rem;
		background: #fff;
	}

	.section-head {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 0.55rem;
	}

	.panel-section {
		padding: 0.8rem;
		border: 1px solid #e2e8f0;
		border-radius: 0.75rem;
		background: #f8fafc;
	}

	.panel-item {
		border: 1px solid #e2e8f0;
		border-radius: 0.65rem;
		background: #fff;
		padding: 0.65rem;
	}

	.panel-item + .panel-item {
		margin-top: 0.6rem;
	}

	.panel-item summary {
		list-style: none;
		cursor: pointer;
		border-radius: 0.45rem;
		padding: 0.2rem 0.25rem;
		transition: background-color 140ms ease, box-shadow 140ms ease;
	}

	.panel-item summary::-webkit-details-marker {
		display: none;
	}

	.panel-item summary:hover {
		background: #f8fafc;
	}

	.panel-item summary:focus-visible {
		outline: none;
		box-shadow: 0 0 0 2px #bfdbfe;
	}

	.panel-summary-row {
		display: flex;
		gap: 0.75rem;
		align-items: center;
		justify-content: space-between;
	}

	.panel-summary-copy {
		display: flex;
		flex-direction: column;
		gap: 0.2rem;
	}

	.panel-title {
		font-weight: 700;
		color: #0f172a;
	}

	.panel-meta {
		font-size: 0.82rem;
		color: #475569;
	}

	.panel-toggle {
		display: inline-flex;
		align-items: center;
		gap: 0.4rem;
		color: #475569;
		font-size: 0.78rem;
		font-weight: 700;
		white-space: nowrap;
	}

	.panel-chevron {
		font-size: 0.95rem;
		line-height: 1;
		transition: transform 160ms ease;
	}

	.panel-item[open] .panel-chevron {
		transform: rotate(180deg);
	}

	.panel-fields {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
		gap: 0.65rem;
		margin-top: 0.75rem;
	}

	.panel-actions {
		margin-top: 0.65rem;
		display: flex;
		justify-content: flex-end;
	}

	.grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
		gap: 0.8rem;
	}

	.advanced {
		padding: 0.8rem;
		border: 1px solid #e2e8f0;
		border-radius: 0.75rem;
		background: #f8fafc;
	}

	.advanced summary {
		cursor: pointer;
		font-weight: 700;
		margin-bottom: 0.8rem;
	}

	button {
		font: inherit;
		font-weight: 700;
		padding: 0.7rem 1rem;
		border: none;
		border-radius: 0.7rem;
		background: linear-gradient(135deg, #0ea5e9, #22c55e);
		color: #fff;
		cursor: pointer;
	}

	button:disabled {
		opacity: 0.6;
		cursor: wait;
	}

	button.ghost,
	button.remove {
		background: #e2e8f0;
		color: #0f172a;
		padding: 0.55rem 0.75rem;
	}

	button.remove:disabled {
		cursor: not-allowed;
		opacity: 0.5;
	}

	.error {
		margin: 1rem 0 0;
		padding: 0.8rem;
		border-radius: 0.7rem;
		background: #fee2e2;
		color: #991b1b;
		font-weight: 600;
	}

	.results {
		margin-top: 0;
	}

	.placeholder {
		flex: 1;
		min-height: clamp(360px, 66vh, 860px);
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		text-align: center;
		background:
			radial-gradient(circle at 20% 10%, rgba(125, 211, 252, 0.32), transparent 40%),
			radial-gradient(circle at 90% 80%, rgba(196, 181, 253, 0.26), transparent 35%),
			linear-gradient(170deg, #ffffff, #f8fafc);
	}

	.empty-state-icon {
		font-size: clamp(78px, 10vw, 132px);
		line-height: 1;
		margin-bottom: 0.8rem;
		filter: drop-shadow(0 10px 20px rgba(59, 130, 246, 0.22));
	}

	.empty-state-title {
		font-size: clamp(1.5rem, 3vw, 2.2rem);
		margin-bottom: 0.15rem;
	}

	.empty-state-copy {
		max-width: 32ch;
	}

	h2 {
		margin: 0;
	}

	.results-head {
		display: flex;
		justify-content: space-between;
		align-items: flex-start;
		gap: 0.8rem;
		margin-bottom: 0.25rem;
	}

	.meta {
		margin: 0.25rem 0 1rem;
		color: #64748b;
	}

	.totals {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
		gap: 0.7rem;
		margin-bottom: 1rem;
	}

	.totals div {
		padding: 0.8rem;
		border-radius: 0.7rem;
		background: #f1f5f9;
	}

	.totals span {
		display: block;
		font-size: 0.85rem;
		color: #475569;
	}

	.totals strong {
		font-size: 1.1rem;
	}

	.chart-wrap {
		margin: 0.8rem 0 1rem;
		padding: 0.95rem;
		border: 1px solid #e2e8f0;
		border-radius: 0.75rem;
		background: linear-gradient(165deg, #ffffff, #f8fafc);
		box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.8);
		position: relative;
	}

	.legend {
		display: flex;
		flex-wrap: wrap;
		gap: 0.45rem;
		margin-bottom: 0.7rem;
	}

	.chip {
		display: inline-flex;
		align-items: center;
		gap: 0.35rem;
		padding: 0.22rem 0.55rem;
		border-radius: 999px;
		font-size: 0.75rem;
		color: #0f172a;
		background: #f8fafc;
		border: 1px solid #e2e8f0;
	}

	.chip-disabled {
		opacity: 0.4;
		filter: grayscale(0.6);
	}

	.swatch {
		width: 10px;
		height: 10px;
		border-radius: 50%;
		flex-shrink: 0;
	}

	.line-swatch {
		background: transparent;
		border-radius: 0;
		border-top: 2px solid #7c3aed;
		width: 12px;
		height: 0;
	}

	svg {
		width: 100%;
		height: auto;
		display: block;
	}

	.axis {
		font-size: 10px;
		fill: #475569;
	}

	.x-axis-label {
		font-size: 14px;
		font-weight: 700;
	}

	.y-axis-label {
		font-size: 11px;
		font-weight: 700;
	}

	.weather-icon {
		font-size: 39px;
	}

	.chart-tooltip {
		position: absolute;
		transform: translate(0, -100%);
		background: rgba(15, 23, 42, 0.94);
		color: #f8fafc;
		padding: 0.45rem 0.55rem;
		border-radius: 0.45rem;
		font-size: 0.78rem;
		white-space: nowrap;
		pointer-events: none;
		z-index: 4;
		box-shadow: 0 6px 18px rgba(2, 6, 23, 0.25);
	}

	.tooltip-title {
		font-weight: 700;
		margin-bottom: 0.25rem;
	}

	.tooltip-row {
		display: grid;
		grid-template-columns: auto 1fr auto;
		align-items: center;
		gap: 0.35rem;
	}

	table {
		width: 100%;
		border-collapse: collapse;
		font-size: 0.92rem;
	}

	th,
	td {
		padding: 0.55rem;
		text-align: left;
		border-bottom: 1px solid #e2e8f0;
	}

	.note {
		margin-top: 1rem;
		font-size: 0.85rem;
		color: #475569;
	}

	@media (max-width: 920px) {
		.panel-fields {
			grid-template-columns: repeat(2, minmax(120px, 1fr));
		}
	}

	@media (min-width: 1100px) {
		.app-layout {
			grid-template-columns: minmax(360px, 0.95fr) minmax(0, 1.45fr);
			align-items: start;
		}

		.form-card {
			position: sticky;
			top: 1rem;
		}
	}

	@media (max-width: 720px) {
		.right-col {
			display: block;
		}

		.placeholder {
			min-height: 260px;
			padding-top: 2rem;
			padding-bottom: 2rem;
		}

		.results-head {
			flex-direction: column;
			align-items: stretch;
		}

		.results-head .ghost {
			width: 100%;
		}

		.panel-fields {
			grid-template-columns: 1fr;
		}

		.panel-actions .remove {
			width: 100%;
		}

		table {
			font-size: 0.82rem;
		}

		th,
		td {
			padding: 0.45rem;
		}
	}
</style>
