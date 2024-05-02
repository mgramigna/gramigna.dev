---
title: "Building a Movie Game in a Weekend"
description: "How I built FilmRover, the Wikipedia game for movies"
pubDate: "Mar 30 2024"
---

A few months ago, I was sitting on the T looking at [Letterboxd](https://letterboxd.com/) on my phone (obviously). My usual habit on Letterboxd is to check out the movies that my friends have been reviewing, click into those, click on the cast, click on a specific cast member, see what they've starred in, and repeat. Before I knew it, I was deep into the filmography of an actor I've never heard of, having no idea how I got there. This gave me an idea: what if I gamified this?

Back in high school, my brother and I used to race each other in the "Wikipedia" game, where you have to get from one random article to another only by clicking links. I always found this super fun, and thought it would be a cool idea to do this for movies. What if my **win condition** was to get to said actor I've never heard of by starting at _Past Lives_ (masterpiece by the way). Thus, the idea for [FilmRover](https://filmrover.vercel.app/) was born.

# The Idea

The idea for an MVP of FilmRover was pretty simple:

- Have an starting movie/actor, and a destination movie/actor
- The filmographies and credits show up in the app, and you can click any "link" to navigate between movies and actors
- A timer is counting up, and stops when you get to the end and win

If I could get the above 3 items in place as a full stack web application, I would be satisfied. But naturally, I had to think of some enhancements that would be really cool to add if I ever had the time and motivation:

- Some kind of leaderboard
- Daily challenges where the start/end is selected for you, updated automatically
- Live challenges, where you can race against another player, rather than just the clock

# The Tech

I already had a tech stack in mind when I thought of the idea. I've worked a lot with the [tMDB API](https://developer.themoviedb.org/docs/getting-started) in the past. It's fast, free, and has a nice collection of endpoints that should cover everything I need. Then, as usual, I'll reach for [Nextjs](https://nextjs.org/)/[TailwindCSS](https://tailwindcss.com/)/[Shadcn UI](https://ui.shadcn.com/) for the frontend.

I also decided to set up a super lightweight database to hold basic information about each "game" someone plays, using [Drizzle ORM](https://orm.drizzle.team/) and an SQLite databse deployed to [Turso](https://turso.tech/).

By throwing all of this tech up on my Vercel hobby account, yet again, I seem to have found a way to deploy a production-ready web application fit for many users completely for free!

# Repository Setup

I've been using [Turborepo](https://turbo.build/repo) more and more on personal projects. It's been helping with both the separation of concerns with various packages. It's easy and fast to work with, and deploys well to Vercel. My go-to is the [create-t3-turbo](https://github.com/t3-oss/create-t3-turbo) template, which not only sets me up with Turborepo, but also some libraries I regularly use:

- A Nextjs app
- Shared TailwindCSS config, eslint config, and prettier config
- A shared `api` package with type safe API calls using [tRPC](https://trpc.io)

Here's the full skeleton of the repo after the initial setup and cleaning up some boilerplate

```
.
├── apps
│   └── nextjs     --> Nextjs application with React
├── packages
│   └── api        --> tRPC API routes
├── tooling
│   ├── eslint     --> Shared eslint config
│   ├── github     --> Shared github actions setup and config
│   ├── prettier   --> Shared prettier config
│   ├── tailwind   --> Shared tailwind config
│   └── typescript --> Shared tsconfig files
└── turbo
    └── generators --> Utility for creating new packages
```

Most of the tooling and otherwise is "set it and forget it." The work is largely scoped to `packages` and `apps`.

# Getting Started with tMDB

The first thing I usually do when starting a project like this is try to tackle the backend. That way, I don't spend too much time up front stressing the petty design details until I'm confident about how the data gets there in the first place.

I started by identifying which API routes the tMDB API has that could get me what I needed, and I got started with these:

- [`/movie/[id]`](https://developer.themoviedb.org/reference/movie-details) - Get the details of a specific movie
- [`/movie/[id]/credits`](https://developer.themoviedb.org/reference/movie-credits) - Get the "credits" for a movie (i.e. cast and crew)
- [`/person/[id]`](https://developer.themoviedb.org/reference/person-details) - Get the details of a specific person
- [`/person/[id]/movie_credits`](https://developer.themoviedb.org/reference/person-movie-credits) - Get all the movie credits for a specific person

This got me exactly what I needed, and enabled me to link people to movies and vice versa

## Data Model

After identifying the tMDB API routes I needed to get started, I took a look at their docs and started thinking about a way to interact with them in a type-safe manner. I've used a few tMDB Node clients in the past, but none of them have had great TypeScript support, and I didn't need the full API support, just the above four API routes for now.

My plan was to use [zod](https://zod.dev) to define the basic structures for movies and people, then make a basic tMDB client that uses these schemas to parse and validate the responses.

Here is the general structure of a movie, which I got from looking at the [docs](https://developer.themoviedb.org/reference/movie-details) for the response to a movie request:

```ts
export const MovieDetailSchema = z.object({
  id: z.number(),
  poster_path: z.string().nullable(),
  release_date: z.union([
    z.literal(""),
    z.string().regex(/\d\d\d\d-\d\d-\d\d/),
  ]),
  title: z.string(),
  // Other fields ommitted
});
```

Most of these fields don't matter, but it was easy enough to copy them over from the docs and see if any of them would come in handy later.

```ts
export const PersonDetailSchema = z.object({
  id: z.number(),
  name: z.string(),
  profile_path: z.string().nullable(),
  // Other fields omitted
});
```

I repeated this for the calls to `/credits` and `/movie_credits`

## Handling the Responses

Defining the schemas is all fine and good, but I also needed to create a simple client and add a way to interact with the API and use these schemas in order to parse the details. My strategy for this was to define a `fetch` helper that takes in a `zod` schema, sends a request to the tMDB API, and parses that result into a type safe result using the schema.

Here's the skeleton code:

```ts
export class TMDBClient {
  baseUrl: string;

  constructor(public bearerToken: string) {
    this.baseUrl = TMDB_API_URL;
  }

  private fetch<T>({
    schema,
    path,
    query,
  }: {
    schema: ZodType<T, ZodTypeDef, T>;
    path: string;
    query?: string;
  }) {
    // ...
  }
}
```

But how did I handle errors? What am I going to do, let an unhandled `Promise` rejection bubble up uncaught all the way through my application? What I am, a monster? Allow me to go on a brief tangent about [neverthrow](https://github.com/supermacro/neverthrow).

### Neverthrow

For anyone that's written [Rust](https://www.rust-lang.org/) code before, you probably love Rust's [`Result`](https://doc.rust-lang.org/std/result/) type. The error handling in that language is super nice. You can match on `Result`, "unwrap" known good results, and handle errors as values rather than wrapping an entire function call in a `try/catch`. Thankfully, a lot of really awesome people have been working hard to port over some of these kinds of language features into TypeScript.

The [neverthrow](https://github.com/supermacro/neverthrow) library is one I've been using a lot more recently, as it allows me to easily treat errors as values in TypeScript, and be way more intentional about the kinds of errors I have in my code.

So now, I can make use of [`ResultAsync.fromPromise`](https://github.com/supermacro/neverthrow?tab=readme-ov-file#resultasyncfrompromise-static-class-method) to wrap my fetch call in a `Result` type. Never throw!

```ts
export class TMDBClient {
  // ...

  private fetch<T>({
    schema,
    path,
    query,
  }: {
    schema: ZodType<T, ZodTypeDef, T>;
    path: string;
    query?: string;
  }): ResultAsync<T, Error> {
    const queryString = query ? `?${query}` : "";

    return ResultAsync.fromPromise(
      fetch(`${this.baseUrl}/${path}${queryString}`, {
        method: "GET",
        headers: {
          Authorization: `Bearer ${this.bearerToken}`,
        },
      })
        .then(async (response) => {
          if (!response.ok) {
            throw new Error(
              `${response.status} ${
                response.statusText
              } - ${await response.text()}`,
            );
          }

          return response.json();
        })
        .then((data: unknown) => schema.parse(data)),
      (e) => {
        if (e instanceof Error) {
          return new Error(e.message);
        }

        return new Error("Unknown error occurred querying tmdb API");
      },
    );
  }
}
```

To summarize, the above method works as follows:

1. Make a request to the tMDB API, optionally with query params
2. Handle the fetch response, re-throwing any errors that occur along the way
3. Parse the JSON response with the provided `zod` schema
4. If any error occurs along the way, **treat that error as a value on the `Result` type**

## Putting it All together

Almost there, I promise. I'll get to the movie game in a bit. I have my models for movies and people, I have a way to type-safely interact with the tMDB API, so all I had to do was write a few methods for doing so along with the tRPC routes to accompany them:

```ts
export class TMDBClient {
  // ...

  async getMovieById({
    id,
  }: {
    id: number;
  }): Promise<Result<MovieDetail, Error>> {
    // Result is automatically inferred as type Result<MovieDetail, Error>
    const result = await this.fetch({
      schema: MovieDetailSchema,
      path: `/movie/${id}`,
    });

    return result;
  }

  async getPersonById({
    id,
  }: {
    id: number;
  }): Promise<Result<PersonDetail, Error>> {
    // Result is automatically inferred as type Result<PersnDetail, Error>
    const result = await this.fetch({
      schema: PersonDetailSchema,
      path: `/person/${id}`,
    });

    return result;
  }
}
```

That's the gist of the tMDB client. If I needed to add more handling for other API routes (spoiler: I did), I could just define the `zod` schema, write the fetch call, and let the type-safety do its thing!

The only other thing I'll cover here is an example of a tRPC router I wrote that interacts with this client. Here's a snippet of the movie router. The person router looks very similar:

```ts
export const movieRouter = createTRPCRouter({
  getById: publicProcedure
    .input(
      z.object({
        id: z.number(),
      }),
    )
    .query(async ({ input, ctx }) => {
      // I automatically attached the tmdb client to tRPC context
      const result = await ctx.tmdb.getMovieById({
        id: input.id,
      });

      if (result.isErr()) {
        throw new TRPCError({
          code: "INTERNAL_SERVER_ERROR",
          message: result.error.message,
          cause: result.error,
        });
      }

      // The type of result is narrowed by `neverthrow`
      // `result.value` is always type `MovieDetail` here
      return result.value;
    }),
});
```

# The Database

All the above took a few hours to get right, but the motivation did not stop. My cat is long asleep for the night at this point, but I wanted to keep plugging away.

Now that I had a way to get the data I cared about, I had to start thinking of the concept of a **game**. A **game** in this application will have a start, a destination, and a state for whether the game has ended or not. It is not a very complex data model, but I liked adding a database anyway to allow for as many creations of custom games as the server allows. Most trendy "daily challenge" games I've seen limit you to just one play per day. That's not what I was interested in building here. So, let's add a database!

I started by adding a new `packages/db` package to the monorepo with `turbo gen init`. This uses the pre-existing `turbo` template we defined for adding a new package. Next, I set up the actual database and the ORM to connect to it.

## Enter Turso

With such a simple data model, using sqlite just made sense to me. I looked to [Turso](https://turso.tech/) for the first time. I had heard about it before, but never had a project that needed it. It has an incredibly generous free tier (500 databases!!), and is super easy to get started with.

Just look at their [quickstart guide](https://docs.turso.tech/quickstart), you can sign up and create a database instantly just by using their CLI.

## Enter Drizzle

This is the second project where I've reached for [Drizzle](https://orm.drizzle.team/), and I've enjoyed my experience with it so far. With any other type-safe ORMs or query builders I've used, it involved some kind of codegen to keep the types up to date. With Drizzle, I can just define my entire schema and types directly in TypeScript. It also lets me decide if I want to use a "SQL-like" query syntax, or a more traditional ORM-style "relational" syntax (similar to [Prisma](https://www.prisma.io/)).

Once I got my database and auth token set up with Turso, I could easily connect it to Drizzle:

```ts
import { createClient } from "@libsql/client";
import { drizzle } from "drizzle-orm/libsql";

const client = createClient({
  url: process.env.DATABASE_URL!,
  authToken: process.env.DATABASE_AUTH_TOKEN!,
});

export const db = drizzle(client);

export * from "drizzle-orm";
export * from "./schema";
```

So what is `./schema`? Well, it's just the defined database schema that I can do directly in TypeScript with Drizzle!

```ts
import { sql } from "drizzle-orm";
import { int, sqliteTable, text } from "drizzle-orm/sqlite-core";

export const games = sqliteTable("games", {
  id: text("id").primaryKey(),
  createdAt: text("created_at")
    .notNull()
    .default(sql`CURRENT_TIMESTAMP`),
  startMovieId: int("start_movie_id"),
  endMovieId: int("end_movie_id"),
  startPersonId: int("start_person_id"),
  endPersonId: int("end_person_id"),
  isFinished: int("is_finished", { mode: "boolean" }),
});
```

## Connecting the Database to tRPC

I decided to put the drizzle client directly on the tRPC context, so I could use it within the routers. The last step remaining here was just to add some basic CRUD to a tRPC router for games. I won't include all the code here, but here's the gist:

```ts
export const gameRouter = createTRPCRouter({
  getById: publicProcedure
    .input(
      z.object({
        id: z.string(),
      }),
    )
    .query(async ({ input, ctx }) => {
      const [result] = await ctx.db
        .select()
        .from(games)
        .where(eq(games.id, input.id));
      // ...
    }),
  list: publicProcedure.query(async ({ ctx }) => {
    const result = await ctx.db.select().from(games);
    // ...
  }),
  create: publicProcedure
    .input(
      z.object({
        // ...
      }),
    )
    .mutation(async ({ ctx, input }) => {
      // ...
      const [result] = await ctx.db
        .insert(games)
        .values({
          id,
          // ...
        })
        .returning({
          gameId: games.id,
        });
      // ...
    }),
  complete: publicProcedure
    .input(
      z.object({
        gameId: z.string(),
      }),
    )
    .mutation(async ({ ctx, input }) => {
      // ...
      await ctx.db
        .update(games)
        .set({
          isFinished: true,
        })
        .where(eq(games.id, input.gameId));
      // ...
    }),
});
```

# Taking a Breath

According to my commit timestamps, it was 11PM at this point. I was just getting started. This was a good time to take a break though. To recap:

- I have a way to interact with the tMDB API
- I have a way to create, read, and update games in a serverless sqlite database that's already deployed
- I am not tired yet

As I reflect on writing this, I'm starting to realize how long this post is going to be. Oh well, time for the frontend!

# Starting the Frontend
