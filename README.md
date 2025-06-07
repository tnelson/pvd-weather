# Weather App (React) 

This is an experiment in using Github Copilot (with GPT-4.1) to generate a weather application. 

## Notes on Iterative Prompting

It produced something that actually ran the first time. I only had to import a library it had missed, but its code "worked". (I didn't get the first couple of prompts saved, unfortunately. But the prompt said to use the NWS API and to create a component that displayed the weather in Providence, RI.)

The trouble is that it printed only the first period forecast, which (at the time, it was around 1:30pm) was for overnight. So, on a hot summer day, it told me that it was under 70 degrees (F) out. I couldn't get it to refine that initial version correctly with more high-level prompting. So I gave a low-level one:

> data.properties.periods contains an array of objects, each of which defines a weather forecast for a specific period. I would like the contents of each of those objects to be shown on the page. Use an HTML table to format them, with one object per row.

This produced something workable, again in one iteration. But this low-level prompt would have been difficult to give without knowledge of the API. (I was using GPT4.1, but tried o1 as well.)

Something else worth noting:
  * The changes left in some unused code it had previously created, in particular the type definition of a weather forecast object. So it's not cleaning up after itself, and not using TypeScript well either. In fact, I'm now curious how much it will lean on all that plain JavaScript in its training data. Notice stuff like: `const [periods, setPeriods] = useState<any[] | null>(null);`. 
  * It used a lot of chained promises instead of `async` and `await`, and its error handling is odd. It nicely reports error responses and has a custom message if the data is still being loaded. But it also added a `if (!periods) return null;` line, which I'm not sure will ever run. (Once the `useEffect` finishes, either `periods` will be populated or `error ` will be. And before it finishes, `loading` is true.)


> Please refine the WeatherData type into an interface that accurately represents the data within a single forecast object, and rename it appropriately.

Here it created the type, but didn't use it. 

> Please add appropriate type definitions using the ForecastPeriod type.

It added some type info, but is now using `as`: 

```
setPeriods(data.properties.periods as ForecastPeriod[]);
```

> Please don't use `as` for typecasting. Instead, create a TypeScript type predicate for the type, and check it at runtime, allowing narrowing to do its job.

This is interesting. It *removed* the type of the forecast record being mapped over. It's also using `unknown`. I would have used `any` here, but `unknown` is a better choice. There's no risk of `any` not being properly handled outside the type predicate.

```
// Type predicate for ForecastPeriod[]
function isForecastPeriodArray(arr: unknown): arr is ForecastPeriod[] {
    return Array.isArray(arr) && arr.every(isForecastPeriod);
}
```

It didn't do badly at explaining why (although it was pretty verbose):

> Why use unknown in the type predicate?

Using unknown in a type predicate like function isForecastPeriod(obj: unknown): obj is ForecastPeriod is a TypeScript best practice because:

Safety: unknown is the safest type for untrusted data (such as API responses). It forces you to explicitly check the structure and types of the value before treating it as a specific type.
Type narrowing: Type predicates (obj is ForecastPeriod) allow TypeScript to narrow the type after the check, so you can safely use the object as a ForecastPeriod only if it passes the predicate.
Avoids unsafe assumptions: If you used any, TypeScript would not enforce any checks, and you could accidentally use properties that don't exist or have the wrong type, leading to runtime errors.
Summary:
Using unknown ensures you validate the data before using it, making your code safer and more robust.

But one issue is that it's casting with `as` inside the type predicate:

```
function isForecastPeriod(obj: unknown): obj is ForecastPeriod {
    if (typeof obj !== 'object' || obj === null) return false;
    const o = obj as Record<string, unknown>;
...
```

in order to get at the fields. 

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

But this is a strange explanation. 

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

So it's "forgotten" the reason to use `unknown` and isn't really getting my hints.

> Please remove the declaration of `o`, it's unused. Also, please stop typecasting `obj` to `any`. Why are you doing that?

It removed the `o` declaration, replacing it with `// No need to declare 'o' or use 'as any'; instead, use type assertion with 'obj' as a record for property access.`. But it isn't doing that. And it kept the casts to `any` in.

```

It replied that `as any` is not necessary and less type-safe (than what?) then suggested that we go back to the `o` approach. "This approach is safer than as any because it preserves some type checking and avoids bypassing TypeScript’s safety features.". 

> Please just replace `obj as any` with `obj` and confirm it still typechecks.

"If you simply replace (obj as any).property with obj.property, TypeScript will give you an error because obj is of type object, which does not guarantee the presence of any properties.

The correct and type-safe way is to use a type assertion to Record<string, unknown>, ..."

So I told it to just make the change. Then, I asked:

> I'm not getting the type errors you said I would get. Why do you think I'm not getting any errors?

"Great observation! You are not getting type errors because of how TypeScript handles property access with the in operator:

When you check if ('number' in obj), TypeScript narrows the type of obj within that block. ..."

Now let's see if it can help me clean this up.

> Thanks. Can we combine each field check (the `in` and the `typeof` parts) together so that we have a list of `return false` conditions, one for each field? 

The first solution it gave me uses `obj as any` again... then it reminded me: "f you want to avoid as any, you can use a type assertion to Record<string, unknown>: ..."

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




