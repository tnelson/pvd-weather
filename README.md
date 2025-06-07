# Weather App (React) 

This is an experiment in using Github Copilot (with GPT-4.1) to generate a weather application. 

Concretely, this is a React application using TypeScript and Vite. 

## Summary

In general, Copilot was helpful for some things (indeed, it surprised me once). Other things I found it very difficult to get done, and either needed to prompt at a lower level or very specifically tell it to do something that it didn't "want" to do. It took around 2 hours of work, and used 20 percent of my free Copilot credits. 

## Notes on Iterative Prompting

### Initial Effort

I didn't get the first couple of prompts saved, unfortunately. But the prompt said to use the NWS API and to create a component that displayed the weather in Providence, RI.

It produced something that actually ran the first time. I only had to import a library it had missed, but its code "worked" and ran in dev mode. The trouble is that it printed only the *first period* forecast, when the NWS returns multiple periods that cover the next week. At the time, it was around 1:30pm but the first period was for overnight. So, on a hot summer day, it told me that it was under 70 degrees (F) out. 

I couldn't get it to refine that initial version correctly with more high-level prompting. So I gave a low-level one:

> data.properties.periods contains an array of objects, each of which defines a weather forecast for a specific period. I would like the contents of each of those objects to be shown on the page. Use an HTML table to format them, with one object per row.

This produced something workable, again in one iteration. But this low-level prompt would have been difficult to give without knowledge of the API. (I was using GPT4.1, but tried o1 as well.)

Also worth noting:
  * The changes left in some unused code it had previously created (the type definition of a weather forecast object). So it's not cleaning up after itself, and not using TypeScript well either (unless I prompt it to). Is it leaning on all that plain JavaScript in its training data? Notice stuff like: `const [periods, setPeriods] = useState<any[] | null>(null);`. 
  * It used a lot of chained promises instead of `async` and `await`, and its error handling is odd if OK. It nicely reports error responses and has a custom message if the data is still being loaded. But it also added a `if (!periods) return null;` line, which I'm not sure will ever run. (Once the `useEffect` finishes, either `periods` will be populated or `error ` will be. And before it finishes, `loading` is true.)

At this point, I jumped into the missing types.

### Adding TypeScript Types

> Please refine the WeatherData type into an interface that accurately represents the data within a single forecast object, and rename it appropriately.

Here it created the type, but didn't use it. 

> Please add appropriate type definitions using the ForecastPeriod type.

It added some type info, but is now using `as`: 

```
setPeriods(data.properties.periods as ForecastPeriod[]);
```

> Please don't use `as` for typecasting. Instead, create a TypeScript type predicate for the type, and check it at runtime, allowing narrowing to do its job.

This is interesting. It *removed* the type of the forecast record being mapped over. It's also using `unknown`. I would have used `any` here, but `unknown` is actually a better choice (and probably reveals a bad habit of mine; `unknown` [was added in 3.0](https://devblogs.microsoft.com/typescript/announcing-typescript-3-0/#the-unknown-type), which was quite a while ago, but less used in examples when I learned TypeScript). 

Here it's better to use `unknown`. There's no risk of `any` not being properly handled outside the type predicate.

```
// Type predicate for ForecastPeriod[]
function isForecastPeriodArray(arr: unknown): arr is ForecastPeriod[] {
    return Array.isArray(arr) && arr.every(isForecastPeriod);
}
```

It didn't do badly at explaining why `unknown` is better (although it was pretty verbose).

But then I noticed an issue. It's casting with `as` inside the type predicate in order to get at the fields:

```
function isForecastPeriod(obj: unknown): obj is ForecastPeriod {
    if (typeof obj !== 'object' || obj === null) return false;
    const o = obj as Record<string, unknown>;

...

}
```

> isForecastPeriod is using casting with any in order to access the fields of the object. Is there a better way to do this? I think perhaps we could use the "in" operator?

It rewrote the predicate to check membership of the fields, but then did the same thing after that...

```
// Type predicate for ForecastPeriod
function isForecastPeriod(obj: unknown): obj is ForecastPeriod {
    if (typeof obj !== 'object' || obj === null) return false;
    if (
        !('number' in obj) ||
        !('name' in obj) ||
        !('startTime' in obj) ||
        !('endTime' in obj) ||
        !('isDaytime' in obj) ||
        !('temperature' in obj) ||
        !('temperatureUnit' in obj) ||
        !('temperatureTrend' in obj) ||
        !('windSpeed' in obj) ||
        !('windDirection' in obj) ||
        !('icon' in obj) ||
        !('shortForecast' in obj) ||
        !('detailedForecast' in obj)
    ) {
        return false;
    }
    const o = obj as Record<string, unknown>;
    return (
        typeof o.number === 'number' &&
        typeof o.name === 'string' &&
        typeof o.startTime === 'string' &&
        typeof o.endTime === 'string' &&
        typeof o.isDaytime === 'boolean' &&
        typeof o.temperature === 'number' &&
        typeof o.temperatureUnit === 'string' &&
        (typeof o.temperatureTrend === 'string' || o.temperatureTrend === null) &&
        typeof o.windSpeed === 'string' &&
        typeof o.windDirection === 'string' &&
        typeof o.icon === 'string' &&
        typeof o.shortForecast === 'string' &&
        typeof o.detailedForecast === 'string'
    );
}
```

> Now that the presence of these fields is confirmed, I'd like to check their types without typecasting with as.

It replaced the typecast with this:

```
    // Use a type guard to ensure obj is an object with string keys
    if (typeof obj !== 'object' || obj === null) return false;
    const o: { [key: string]: unknown } = obj;
```

> I don't think we need o at all, and could use obj directly.

Now it's left `o` in, but unused, and added a bunch of casts to `any`. 

```
const o: { [key: string]: unknown } = obj;
    return (
        typeof (obj as any).number === 'number' &&
        typeof (obj as any).name === 'string' &&
        typeof (obj as any).startTime === 'string' &&
        typeof (obj as any).endTime === 'string' &&
        typeof (obj as any).isDaytime === 'boolean' &&
        typeof (obj as any).temperature === 'number' &&
        typeof (obj as any).temperatureUnit === 'string' &&
        (typeof (obj as any).temperatureTrend === 'string' || (obj as any).temperatureTrend === null) &&
        typeof (obj as any).windSpeed === 'string' &&
        typeof (obj as any).windDirection === 'string' &&
        typeof (obj as any).icon === 'string' &&
        typeof (obj as any).shortForecast === 'string' &&
        typeof (obj as any).detailedForecast === 'string'
    );
```

It isn't really getting my hints.

> Please remove the declaration of `o`, it's unused. Also, please stop typecasting `obj` to `any`. Why are you doing that?

It removed the `o` declaration, replacing it with `// No need to declare 'o' or use 'as any'; instead, use type assertion with 'obj' as a record for property access.`. But it isn't doing the record thing anymore. And it kept the casts to `any` in.

It replied that `as any` is not necessary and less type-safe (than what?) then suggested that we go back to the `o` approach with records. "This approach is safer than as any because it preserves some type checking and avoids bypassing TypeScript’s safety features.". 

> Please just replace `obj as any` with `obj` and confirm it still typechecks.

"If you simply replace (obj as any).property with obj.property, TypeScript will give you an error because obj is of type object, which does not guarantee the presence of any properties.

The correct and type-safe way is to use a type assertion to Record<string, unknown>, ..."

So I told it to just make the change. Then, I asked:

> I'm not getting the type errors you said I would get. Why do you think I'm not getting any errors?

"Great observation! You are not getting type errors because of how TypeScript handles property access with the in operator:

When you check if ('number' in obj), TypeScript narrows the type of obj within that block. ..."

Now let's see if it can help me clean this up.

> Thanks. Can we combine each field check (the `in` and the `typeof` parts) together so that we have a list of `return false` conditions, one for each field? 

The first solution it gave me uses `obj as any` again... then it reminded me: "If you want to avoid as any, you can use a type assertion to Record<string, unknown>: ..."

> No, I don't want to do either of those things. I want to confirm each field's presence and its type in one if statement. Like this: `if (! 'name" in obj || typeof obj.name !== 'string') return false`. Can you do this?

It did, except with `obj as any`. Sigh. So I asked it to just remove the typecast again. 

Now let's try formatting:

> Instead of putting the temperature and the temperature unit in separate columns, can you combine them into one column that just concatenates the strings, so we'd see something like "70 F"?

This worked right the first time. 

Great! Now let's see if it can help me deploy this on Github pages.

> This looks great! Can you help me to deploy this web application on Github Pages? 

It gave me a series of instructions, which I am condensing here. 

- Add this to `package.json`: `"homepage": "https://your-username.github.io/your-repo"` (replacing the names). 
- Run `npm install --save-dev gh-pages`
- Add to the `scripts` in `package.json`: `"predeploy": "npm run build",
"deploy": "gh-pages -d build"`.
- Push to Github 
- npm run deploy
- Go to your repository’s Settings > Pages and set the source branch to gh-pages (if it’s not set automatically).
- Visit the app at `https://your-username.github.io/your-repo`. 

(And it added a warning about React Router.)

It didn't warn me I'd need to wait for the build after enabling Pages, but that's OK. 

But I'm getting errors: it days it can't find `main.tsx`...

Ah, I selected the `main` branch in Github, because there wasn't another option, but I needed to select the `gh-pages` branch (which doesn't exist). 

> gh-pages runs and uploads but it doesn't create a separate branch with the built code in it. How can I fix this?

Its answer was kinda confusing, with lots of things to try. But then I remembered I hadn't run `npm run deploy`. The trouble _now_ was that the branch name didn't match. So I needed to change it from `gh-pages -d build` to `gh-pages -d dist`. (It didn't tell me to `predeploy` but I did.)

Still having issues, likely because of a directory-depth problem (the browser is looking for `tnelson.github.io/assets/...` for this app, which is missing the nested project name). Can Copilot help? (I know this is because of Vite.)

> I think that the browser is looking in the wrong folder for the js files. It says that `https://tnelson.github.io/assets/index-DKdARN1f.js` isn't found (404 error). What could be causing the problem?

It did indeed say that, if using Vite, I needed to add the `base` field in `vite.config.js`.

And now the application [runs](https://tnelson.github.io/pvd-weather).