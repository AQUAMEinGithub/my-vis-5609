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

    let selectedGenre: string = $state();

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

    const entriesSorted: Array<[string, number]> = $derived(
        Object.entries(genreNums).sort((a, b) => d3.descending(a[1], b[1])),
    );
    const genresSorted: string[] = $derived(entriesSorted.map(([g]) => g));
    const maxCount: number = $derived(d3.max(entriesSorted, (d) => d[1]) ?? 0);

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
    function updateAxis() {
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
    }
    $effect(() => {
        updateAxis();
    });
</script>

<h3>
    The Distribution of Genres {yearRange[0]?.getFullYear()} - {yearRange[1]?.getFullYear()}
</h3>

{#if movies.length > 0}
    <svg {width} {height}>
        <g class="bars">
            {#each entriesSorted as [genre, cnt]}
                <g class={genre}>
                    <rect
                        width={xBarwidth}
                        height={yScale(0) - yScale(cnt)}
                        x={xScale(genre)!}
                        y={yScale(cnt)}
                        class="bar"
                        opacity={selectedGenre
                            ? selectedGenre === genre
                                ? 1
                                : 0.35
                            : 1}
                        on:mouseover={() => {
                            selectedGenre = genre;
                        }}
                        on:mouseout={() => {
                            selectedGenre = "";
                        }}
                    />
                    <text
                        x={xScale(genre)! + xBarwidth / 2}
                        y={yScale(cnt) - 6}
                        font-size="12"
                        text-anchor="middle"
                        fill={selectedGenre && selectedGenre !== genre
                            ? "#666"
                            : "#000"}
                        font-weight={selectedGenre === genre ? 700 : 400}
                    >
                        {cnt}
                    </text>
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
    </svg>
{/if}

<style>
    .bar {
        fill: #449900;
        transition:
            y 0.1s ease,
            height 0.1s ease,
            width 0.1s ease,
            opacity 0.1s ease;
        shape-rendering: crispEdges;
    }
</style>
