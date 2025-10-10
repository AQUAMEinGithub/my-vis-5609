<script lang="ts">
    import * as d3 from "d3";
    import type { TMovie } from "../types";

    export let movies: TMovie[] = [];
    export let metric: "count" | "rate" | "lift" = "rate";
    export let topN: number = 5;
    export let size: number = 120;
    export let columns: number = 5;
    export let gap: number = 16;
    export let minAnchorMovies: number = 3;

    const isValid = (g: string) =>
        g &&
        g !== "\\N" &&
        g.toLowerCase() !== "na" &&
        g.toLowerCase() !== "n/a";

    $: genresAll = Array.from(
        new Set(movies.flatMap((m) => (m.genres || []).filter(isValid))),
    ).sort();

    $: countSingle = new Map<string, number>(genresAll.map((g) => [g, 0]));
    $: counts = new Map<string, Map<string, number>>(
        genresAll.map((a) => [a, new Map()]),
    );
    $: totalMovies = movies.length;

    $: {
        for (const g of genresAll) {
            countSingle.set(g, 0);
            const row = counts.get(g)!;
            row.clear();
            for (const h of genresAll) if (h !== g) row.set(h, 0);
        }
        for (const m of movies) {
            const gs = Array.from(new Set((m.genres || []).filter(isValid)));
            for (const a of gs)
                countSingle.set(a, (countSingle.get(a) ?? 0) + 1);
            for (const a of gs) {
                for (const b of gs) {
                    if (a === b) continue;
                    const row = counts.get(a)!;
                    row.set(b, (row.get(b) ?? 0) + 1);
                }
            }
        }
    }

    function valueFor(a: string, b: string): number {
        const cab = counts.get(a)?.get(b) ?? 0;
        if (metric === "count") return cab;
        if (metric === "rate") {
            const ca = countSingle.get(a) ?? 1;
            return cab / ca;
        }
        const ca = countSingle.get(a) ?? 1;
        const cb = countSingle.get(b) ?? 1;
        const p_b = cb / (totalMovies || 1);
        const p_b_given_a = cab / ca;
        return p_b ? p_b_given_a / p_b : 0;
    }

    type Slice = {
        key: string;
        value: number;
        rawCount: number;
        share: number;
    };
    let pies: Record<string, Slice[]> = {};

    $: {
        const next: Record<string, Slice[]> = {};
        for (const a of genresAll) {
            const row = counts.get(a)!;
            const slicesTmp: Omit<Slice, "share">[] = [];
            for (const b of genresAll) {
                if (b === a) continue;
                const raw = row.get(b) ?? 0;
                const v = valueFor(a, b);
                if (raw > 0 || v > 0)
                    slicesTmp.push({ key: b, value: v, rawCount: raw });
            }
            slicesTmp.sort((x, y) => d3.descending(x.value, y.value));
            const top = slicesTmp.slice(0, topN);
            const rest = slicesTmp.slice(topN);
            const otherValue = d3.sum(rest, (s) => s.value);
            const otherCount = d3.sum(rest, (s) => s.rawCount);
            const combined = [...top];
            if (otherValue > 0)
                combined.push({
                    key: "Other",
                    value: otherValue,
                    rawCount: otherCount,
                });

            const totalV = d3.sum(combined, (s) => s.value) || 1;
            next[a] = combined.map((s) => ({ ...s, share: s.value / totalV }));
        }
        pies = next;
    }

    const palette = d3.schemeTableau10;
    $: color = d3
        .scaleOrdinal<string, string>()
        .domain(genresAll)
        .range(palette.concat(palette, palette).slice(0, genresAll.length));

    const otherColor = "#dddddd";
    const radius = size / 2;
    const arcGen = d3
        .arc<any>()
        .innerRadius(0)
        .outerRadius(radius - 2);
    const pieGen = d3
        .pie<Slice>()
        .value((d) => d.share)
        .sort(null);

    let hoverAnchor: string | null = null;
    let hoverKey: string | null = null;

    const fadeOtherSlices = 0.55;
    const fadeOtherPies = 0.12;

    function tipText(a: string, s: Slice): string {
        return `${a} × ${s.key} — ${
            metric === "count"
                ? `${s.rawCount} films`
                : metric === "rate"
                  ? `${(s.value * 100).toFixed(1)}% of ${a}`
                  : `lift ${s.value.toFixed(2)}`
        }`;
    }
</script>

{#if movies.length > 0}
    <div
        class="grid"
        style={`display:grid;grid-template-columns:repeat(${columns}, ${size}px);gap:${gap}px;`}
    >
        {#each genresAll as anchor}
            <div
                class="card"
                style={`width:${size}px;opacity:${hoverAnchor ? (hoverAnchor === anchor ? 1 : fadeOtherPies) : 1};`}
            >
                <div class="title" title={`Anchor: ${anchor}`}>{anchor}</div>

                {#if (countSingle.get(anchor) ?? 0) >= minAnchorMovies}
                    <svg
                        width={size}
                        height={size}
                        role="img"
                        aria-label={`Co-occurrence for ${anchor}`}
                        on:mouseleave={() => {
                            hoverAnchor = null;
                            hoverKey = null;
                        }}
                    >
                        <g transform={`translate(${radius},${radius})`}>
                            {#each pieGen(pies[anchor] ?? []) as a}
                                {@const s = a.data}
                                {@const isHovered =
                                    hoverAnchor === anchor &&
                                    hoverKey === s.key}
                                <g
                                    on:mouseover={() => {
                                        hoverAnchor = anchor;
                                        hoverKey = s.key;
                                    }}
                                >
                                    <path
                                        d={arcGen(a)!}
                                        fill={s.key === "Other"
                                            ? otherColor
                                            : color(s.key)}
                                        opacity={hoverAnchor &&
                                        hoverAnchor === anchor
                                            ? isHovered
                                                ? 1
                                                : fadeOtherSlices
                                            : 1}
                                    >
                                        <title>{tipText(anchor, s)}</title>
                                    </path>

                                    {#if isHovered}
                                        {@const c = arcGen.centroid(a)}
                                        {@const cx = c[0]}
                                        {@const cy = c[1]}
                                        <text
                                            class="label"
                                            x={cx}
                                            y={cy - 6}
                                            text-anchor="middle"
                                            font-size="11"
                                            font-weight="700"
                                            fill="#111"
                                        >
                                            {anchor} × {s.key}
                                        </text>
                                        <text
                                            class="label"
                                            x={cx}
                                            y={cy + 10}
                                            text-anchor="middle"
                                            font-size="10"
                                            fill="#111"
                                        >
                                            {metric === "count"
                                                ? `${s.rawCount} films`
                                                : metric === "rate"
                                                  ? `${(s.value * 100).toFixed(1)}%`
                                                  : `lift ${s.value.toFixed(2)}`}
                                        </text>
                                    {/if}
                                </g>
                            {/each}
                        </g>
                    </svg>
                {:else}
                    <svg
                        width={size}
                        height={size}
                        role="img"
                        aria-label={`Placeholder for ${anchor}`}
                    >
                        <g transform={`translate(${radius},${radius})`}>
                            <circle
                                r={radius - 2}
                                fill="none"
                                stroke="#e5e7eb"
                                stroke-width="3"
                                stroke-dasharray="4 3"
                            />
                            <text
                                class="label"
                                text-anchor="middle"
                                dy="-2"
                                font-size="12"
                                fill="#6b7280">low n</text
                            >
                            <text
                                class="label"
                                text-anchor="middle"
                                dy="14"
                                font-size="11"
                                fill="#6b7280"
                            >
                                n={countSingle.get(anchor) ??
                                    0}/{minAnchorMovies}
                            </text>
                        </g>
                    </svg>
                {/if}
            </div>
        {/each}
    </div>

    <div class="legend">
        {#each genresAll as g}
            <span class="legend-item">
                <svg width="12" height="12"
                    ><rect width="12" height="12" fill={color(g)} /></svg
                >{g}
            </span>
        {/each}
        <span class="legend-item">
            <svg width="12" height="12"
                ><rect width="12" height="12" fill="#dddddd" /></svg
            >Other
        </span>
    </div>
{/if}

<style>
    .card {
        display: flex;
        flex-direction: column;
        align-items: center;
        transition: opacity 0.12s linear;
    }
    .title {
        font:
            700 12px/1.2 ui-sans-serif,
            system-ui,
            -apple-system,
            Segoe UI,
            Roboto,
            Arial;
        margin-bottom: 4px;
    }
    .legend {
        display: flex;
        flex-wrap: wrap;
        gap: 10px;
        margin-top: 8px;
        font-size: 12px;
        align-items: center;
    }
    .legend-item {
        display: inline-flex;
        align-items: center;
        gap: 6px;
    }
    .legend-note {
        opacity: 0.7;
        margin-left: 8px;
    }
    svg text {
        font-family:
            ui-sans-serif,
            system-ui,
            -apple-system,
            Segoe UI,
            Roboto,
            Arial;
    }
    .label {
        pointer-events: none;
    } /* prevent hover flicker */
</style>
