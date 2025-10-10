<script lang="ts">
    import * as d3 from "d3";
    import type { TMovie } from "../types";

    export let movies: TMovie[] = [];
    export let width: number = 900;
    export let height: number = 420;
    export let topN: number = 3;

    $: years = Array.from(
        new Set(movies.map((m) => m.year.getFullYear())),
    ).sort((a, b) => a - b);
    $: genresAll = Array.from(new Set(movies.flatMap((m) => m.genres))).sort();

    $: countsByYear = years.map((y) => {
        const obj: Record<string, number | string> = { year: String(y) };
        genresAll.forEach((g) => (obj[g] = 0));
        movies.forEach((m) => {
            if (m.year.getFullYear() === y) {
                m.genres.forEach((g) => (obj[g] = (obj[g] as number) + 1));
            }
        });
        return obj;
    });

    $: totalsByYear = countsByYear.map((row) =>
        genresAll.reduce((acc, g) => acc + (row[g] as number), 0),
    );
    $: maxTotal = d3.max(totalsByYear) ?? 0;

    $: rankByYear = (() => {
        const map = new Map<number, Map<string, number>>();
        years.forEach((y, idx) => {
            const row = countsByYear[idx];
            const pairs = genresAll
                .map((g) => [g, row[g] as number] as [string, number])
                .sort((a, b) => d3.descending(a[1], b[1]));
            const inner = new Map<string, number>();
            pairs.slice(0, topN).forEach(([g], j) => inner.set(g, j + 1));
            map.set(y, inner);
        });
        return map;
    })();

    $: stack = d3.stack().keys(genresAll)(countsByYear as any);

    const margin = { top: 20, right: 12, bottom: 68, left: 56 };
    $: innerW = width - margin.left - margin.right;
    $: innerH = height - margin.top - margin.bottom;

    $: x = d3
        .scaleBand<string>()
        .domain(years.map(String))
        .range([0, innerW])
        .padding(0.2);
    $: y = d3.scaleLinear().domain([0, maxTotal]).nice().range([innerH, 0]);

    const rankColors = new Map<number, string>([
        [1, "#D4AF37"],
        [2, "#6BAED6"],
        [3, "#FD8D3C"],
    ]);
    const gray = "#dddddd";

    let hoverYear: number | null = null;
    let hoverKey: string | null = null;

    let xAxis: SVGGElement | null;
    let yAxis: SVGGElement | null;
    $: if (xAxis)
        d3.select(xAxis)
            .call(d3.axisBottom(x))
            .selectAll("text")
            .attr("transform", "rotate(45)")
            .style("text-anchor", "start");
    $: if (yAxis)
        d3.select(yAxis).call(
            d3
                .axisLeft(y)
                .ticks(6)
                .tickFormat(d3.format("~s") as any),
        );
</script>

{#if movies.length > 0}
    <svg {width} {height}>
        <g transform={`translate(${margin.left},${margin.top})`}>
            {#each stack as series}
                {#each series as seg, i}
                    {@const y0 = y(seg[0])}
                    {@const y1 = y(seg[1])}
                    {@const yr = years[i]}
                    {@const r = rankByYear.get(yr)?.get(series.key)}
                    {@const val = seg[1] - seg[0]}
                    {@const isSameYear = hoverYear === null || hoverYear === yr}
                    {@const isHoveredSegment =
                        hoverYear === yr && hoverKey === series.key}

                    <rect
                        x={x(String(yr))!}
                        y={y1}
                        width={x.bandwidth()}
                        height={Math.max(0, y0 - y1)}
                        fill={r ? rankColors.get(r)! : gray}
                        opacity={isSameYear ? 1 : 0.25}
                        on:mouseover={() => {
                            hoverYear = yr;
                            hoverKey = series.key;
                        }}
                        on:mouseout={() => {
                            hoverYear = null;
                            hoverKey = null;
                        }}
                    >
                        <title
                            >{`${yr} • ${series.key} • ${r ? `Rank ${r}` : `Not in Top 3`} • ${val.toFixed(0)}`}</title
                        >
                    </rect>

                    {#if isHoveredSegment}
                        <text
                            x={x(String(yr))! + x.bandwidth() / 2}
                            y={Math.max(12, y1 - 6)}
                            text-anchor="middle"
                            font-size="12"
                            font-weight="700"
                            fill="#111"
                        >
                            {yr} • {series.key} • {r
                                ? `Rank ${r}`
                                : `Not in Top 3`} • {val.toFixed(0)}
                        </text>
                    {/if}
                {/each}
            {/each}

            <g transform={`translate(0,${innerH})`} bind:this={xAxis} />
            <g bind:this={yAxis} />

            <text
                x={innerW / 2}
                y={innerH + 48}
                text-anchor="middle"
                font-weight="600">Year</text
            >
            <text
                transform="rotate(-90)"
                x={-innerH / 2}
                y={-40}
                text-anchor="middle"
                font-weight="600">Movies per year (stacked by genre)</text
            >
        </g>
    </svg>

    <div class="legend">
        <span class="legend-item"
            ><svg width="12" height="12"
                ><rect width="12" height="12" fill="#D4AF37" /></svg
            >Rank 1</span
        >
        <span class="legend-item"
            ><svg width="12" height="12"
                ><rect width="12" height="12" fill="#6BAED6" /></svg
            >Rank 2</span
        >
        <span class="legend-item"
            ><svg width="12" height="12"
                ><rect width="12" height="12" fill="#FD8D3C" /></svg
            >Rank 3</span
        >
        <span class="legend-item"
            ><svg width="12" height="12"
                ><rect width="12" height="12" fill="#dddddd" /></svg
            >Not in Top 3</span
        >
    </div>
{/if}

<style>
    .legend {
        display: flex;
        gap: 12px;
        flex-wrap: wrap;
        margin-top: 8px;
        font-size: 12px;
    }
    .legend-item {
        display: inline-flex;
        align-items: center;
        gap: 6px;
    }
    text {
        font-family:
            ui-sans-serif,
            system-ui,
            -apple-system,
            Segoe UI,
            Roboto,
            Arial;
    }
</style>
