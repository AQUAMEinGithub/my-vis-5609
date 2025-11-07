<script lang="ts">
  import type { TMovie } from "../../types";

  type Props = {
    movies: TMovie[];
  };
  let { movies }: Props = $props();

  import { Scroll, Bar } from "$lib";
  import RankBar from "$lib/RankBar.svelte";
  import * as d3 from "d3";
  import { get } from "svelte/store";

  let myProgress = $state(0);

  let yearRange = $derived(d3.extent(movies.map((d) => d.year)));

  function getYearsBetweenDates(startDate?: Date, endDate?: Date) {
    if (startDate && endDate) {
      return Array.from(
        { length: endDate.getFullYear() - startDate.getFullYear() + 1 },
        (_, i) => new Date(startDate.getFullYear() + i, 0, 1),
      );
    }
    return [];
  }
  let years: Date[] = $derived(
    getYearsBetweenDates(yearRange[0], yearRange[1]),
  );

  function getHighestRatedMovieEachYear(
    movies: TMovie[],
    years: Date[] | undefined,
  ): TMovie[] {
    let highestRatedMovieEachYear = years
      ? years.map((currentYear) => {
          let moviesThisYear = movies.filter(
            (d) => d.year.getFullYear() == currentYear.getFullYear(),
          );
          if (moviesThisYear.length === 0) return undefined;
          let highestRatedMovie = moviesThisYear.reduce((max, current) =>
            current.average_rating >= max.average_rating ? current : max,
          );
          return highestRatedMovie;
        })
      : [];
    highestRatedMovieEachYear = highestRatedMovieEachYear.filter((d) => d);
    return highestRatedMovieEachYear;
  }

  let highestRatedMovieEachYear = $derived(
    getHighestRatedMovieEachYear(movies, years),
  );

  const barChaertHeight = 500;
</script>

<h2>Scrolly with 2D visualization</h2>
<p>
  First, we demonstrate how to use the Scrolly element to update the genre
  distribution based on year, which is estimated based on the scrolly y position
  of the left text panel
</p>

<Scroll bind:progress={myProgress}>
  <!-- the above Scroll component also accept variables such as 
  --scrolly-story-width="600px"
  --scrolly-layout="viz-first"
  -->

  <h3>Movie with the highest score for each year</h3>
  {#each highestRatedMovieEachYear as movie}
    <div class={movie.year.getFullYear().toString()}>
      <p>
        <b>{movie.year.getFullYear()}:</b>
        <b class="movie-title">{movie.primary_title}</b>,
        {movie.average_rating}
      </p>
    </div>
  {/each}

  {#each years as date}
    <div class={date.getFullYear().toString()}>
      <h3 class="year">{date.getFullYear()}</h3>
      {#each movies.filter((d) => d.year.getFullYear() == date.getFullYear()) as movie}
        <b class="movie-title">{movie.primary_title}:</b>
        {movie.genres.join(" | ")}
        <br />
      {/each}
    </div>
  {/each}

  <div slot="viz" class="viz-container">
    <Bar {movies} progress={myProgress} height={320} width={480} />
    <RankBar {movies} progress={myProgress} height={300} width={480} />
  </div>

  <style>
    .viz-container {
      display: flex;
      flex-direction: column;
      align-items: flex-start;
      gap: 10px;
      transform: scale(0.9);
      transform-origin: top left;
    }
  </style>
</Scroll>

<style>
  .movie-title {
    color: #449900;
  }
  h3.year {
    margin-bottom: 0px;
  }
</style>
