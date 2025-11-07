<script lang="ts">
    import type { TMovie } from "../types";
    import * as d3 from "d3";

    type Props = {
        movies: TMovie[];
        progress?: number;
        width?: number;
        height?: number;
    };
    let { movies, progress = 100, width = 500, height = 400 }: Props = $props();

    let selectedGenre: string = $state("");

    const yearRange = $derived(d3.extent(movies.map((d) => d.year)));
    function getUpYear(range: [undefined, undefined] | [Date, Date]) {
        if (!range[0] || !range[1]) return new Date();
        const timeScale = d3
            .scaleTime()
            .domain(range as [Date, Date])
            .range([0, 100]);
        return timeScale.invert(progress);
    }
    const upYear: Date = $derived(getUpYear(yearRange!));

    const prevYearEnd: Date = $derived(
        new Date(upYear.getFullYear() - 1, 11, 31),
    );

    function getGenreNums(m: TMovie[], up: Date) {
        const res: Record<string, number> = {};
        m.filter((movie) => movie.year <= up).forEach((movie) => {
            movie.genres.forEach((g) => {
                res[g] = (res[g] || 0) + 1;
            });
        });
        return res;
    }

    const genreNums = $derived(getGenreNums(movies, upYear));
    const prevGenreNums = $derived(getGenreNums(movies, prevYearEnd));

    const entriesSorted: Array<[string, number]> = $derived(
        Object.entries(genreNums).sort((a, b) => d3.descending(a[1], b[1])),
    );
    const genresSorted: string[] = $derived(entriesSorted.map(([g]) => g));
    const maxCount: number = $derived(d3.max(entriesSorted, (d) => d[1]) ?? 0);

    const prevEntriesSorted: Array<[string, number]> = $derived(
        Object.entries(prevGenreNums).sort((a, b) => d3.descending(a[1], b[1])),
    );
    const prevRankMap: Record<string, number> = $derived(
        prevEntriesSorted.reduce(
            (acc, [g], i) => {
                acc[g] = i;
                return acc;
            },
            {} as Record<string, number>,
        ),
    );
    const currRankMap: Record<string, number> = $derived(
        entriesSorted.reduce(
            (acc, [g], i) => {
                acc[g] = i;
                return acc;
            },
            {} as Record<string, number>,
        ),
    );

    type Row = {
        genre: string;
        curr: number;
        prev: number;
        inc: number;
        improved: boolean;
    };
    const rows: Row[] = $derived(
        genresSorted.map((g) => {
            const curr = genreNums[g] || 0;
            const prev = prevGenreNums[g] || 0;
            const inc = Math.max(0, curr - prev);
            const pr = prevRankMap[g];
            const cr = currRankMap[g];
            const improved = pr !== undefined && cr < pr;
            return { genre: g, curr, prev, inc, improved };
        }),
    );

    const margin = { top: 15, bottom: 50, left: 40, right: 10 };
    let usableArea = {
        top: margin.top,
        right: width - margin.right,
        bottom: height - margin.bottom,
        left: margin.left,
    };

    const xScale = $derived(
        d3
            .scaleBand<string>()
            .domain(genresSorted)
            .range([usableArea.left, usableArea.right])
            .padding(0.15),
    );
    const yScale = $derived(
        d3
            .scaleLinear()
            .domain([0, maxCount])
            .nice()
            .range([usableArea.bottom, usableArea.top]),
    );
    const xBarwidth: number = $derived(xScale.bandwidth());

    let xAxis: any = $state(),
        yAxis: any = $state();
    let axisRAF = 0;
    function scheduleAxis() {
        cancelAnimationFrame(axisRAF);
        axisRAF = requestAnimationFrame(() => {
            d3.select(xAxis)
                .call(d3.axisBottom(xScale))
                .selectAll("text")
                .attr("transform", "rotate(45)")
                .style("text-anchor", "start");
            d3.select(yAxis).call(
                d3
                    .axisLeft(yScale)
                    .ticks(6)
                    .tickFormat(d3.format("~s") as any),
            );
        });
    }
    $effect(() => {
        xScale;
        yScale;
        scheduleAxis();
    });
</script>

<h3>
    The Distribution of Genres
    {yearRange[0]?.getFullYear()} - {yearRange[1]?.getFullYear()}
</h3>

{#if movies.length > 0}
    <svg {width} {height}>
        <g class="bars">
            {#each rows as { genre, curr, prev, inc, improved }}
                <g class={genre}>
                    <rect
                        width={xBarwidth}
                        x={xScale(genre)!}
                        y={yScale(prev)}
                        height={yScale(0) - yScale(prev)}
                        class="bar-base"
                        fill={improved ? "#D4AF37" : "#449900"}
                        opacity={selectedGenre
                            ? selectedGenre === genre
                                ? 1
                                : 0.35
                            : 1}
                        on:mouseover={() => (selectedGenre = genre)}
                        on:mouseout={() => (selectedGenre = "")}
                    />
                    <rect
                        width={xBarwidth}
                        x={xScale(genre)!}
                        y={yScale(curr)}
                        height={Math.max(0, yScale(prev) - yScale(curr))}
                        class="bar-inc"
                        fill="#ffb3ab"
                        opacity={selectedGenre
                            ? selectedGenre === genre
                                ? 1
                                : 0.35
                            : 1}
                        on:mouseover={() => (selectedGenre = genre)}
                        on:mouseout={() => (selectedGenre = "")}
                    />
                    <text
                        x={xScale(genre)! + xBarwidth / 2}
                        y={yScale(curr) - 6}
                        font-size="12"
                        text-anchor="middle"
                        fill={selectedGenre && selectedGenre !== genre
                            ? "#666"
                            : improved
                              ? "#7a5f14"
                              : "#000"}
                        font-weight={selectedGenre === genre || improved
                            ? 700
                            : 400}
                    >
                        {curr}
                    </text>
                    {#if inc > 0}
                        <text
                            x={xScale(genre)! + xBarwidth / 2}
                            y={Math.max(usableArea.top + 10, yScale(curr) - 22)}
                            font-size="11"
                            text-anchor="middle"
                            fill="#cc3d00"
                            opacity="0.75"
                            font-weight="600"
                        >
                            +{inc}
                        </text>
                    {/if}
                </g>
            {/each}
        </g>

        <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
        <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />

        <text
            x={(usableArea.left + usableArea.right) / 2}
            y={usableArea.bottom + 36}
            text-anchor="middle"
            font-weight="600">Genre</text
        >
        <text
            transform="rotate(-90)"
            x={-(usableArea.top + usableArea.bottom) / 2}
            y={usableArea.left - 28}
            text-anchor="middle"
            font-weight="600">Number of Movies</text
        >
        <g transform="translate({usableArea.right - 170}, {usableArea.top})">
            <rect width="14" height="14" fill="#449900" opacity="0.9" />
            <text x="20" y="12" font-size="12">Accumulated</text>
            <rect y="20" width="14" height="14" fill="#ffb3ab" opacity="0.65" />
            <text x="20" y="32" font-size="12">Newly added</text>
            <rect y="40" width="14" height="14" fill="#D4AF37" opacity="0.95" />
            <text x="20" y="52" font-size="12">Rank increasing</text>
        </g>
    </svg>
{/if}

<style>
    .bar-base {
        transition:
            y 0.45s ease,
            height 0.45s ease,
            opacity 0.2s ease,
            fill 0.35s ease;
        shape-rendering: crispEdges;
    }
    .bar-inc {
        opacity: 0.65;
        transition:
            y 0.45s ease,
            height 0.45s ease,
            opacity 0.2s ease;
        shape-rendering: crispEdges;
    }
</style>
