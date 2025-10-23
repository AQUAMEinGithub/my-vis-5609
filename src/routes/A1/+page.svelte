<script lang="ts">
    import * as d3 from "d3";
    import { onMount } from "svelte";
    import type { TMovie } from "../../types";

    import Bar from "$lib/Bar.svelte";
    import StackedBars from "$lib/StackedBars.svelte";
    import GenreCooccurrencePies from "$lib/GenreCooccurrencePies.svelte";

    let movies: TMovie[] = [];

    function toNum(v: unknown): number {
        const n = Number(v);
        return Number.isFinite(n) ? n : NaN;
    }

    function splitGenres(v: unknown): string[] {
        const s = String(v ?? "").trim();
        return s
            ? s
                  .split(",")
                  .map((x) => x.trim())
                  .filter(Boolean)
            : [];
    }

    async function loadCsv() {
        try {
            const csvUrl = "summer_movies.csv";
            movies = await d3.csv(csvUrl, (row: any): TMovie | undefined => {
                const yearNum = toNum(row.year);
                if (!Number.isFinite(yearNum)) return undefined;
                return {
                    tconst: row.tconst,
                    title_type: row.title_type,
                    primary_title: row.primary_title,
                    original_title: row.original_title,
                    year: new Date(yearNum, 0, 1),
                    runtime_minutes: toNum(row.runtime_minutes),
                    genres: splitGenres(row.genres),
                    average_rating: toNum(row.average_rating),
                    num_votes: toNum(row.num_votes),
                };
            });
            console.log("Loaded CSV Data:", movies);
        } catch (error) {
            console.error("Error loading CSV:", error);
        }
    }

    onMount(loadCsv);
</script>

<h1>Summer Movies</h1>
<p>Here are {movies.length === 0 ? "..." : movies.length + " "} movies</p>

<Bar {movies} />

<h2>Q1 · Stacked Bars (Top 3 highlighted)</h2>
<StackedBars {movies} />

<h2>Q2 · Genre Co-occurrence (Top N slices per anchor)</h2>
<GenreCooccurrencePies {movies} metric="rate" topN={5} size={120} columns={5} />
