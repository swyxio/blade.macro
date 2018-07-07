# blade.macro

this is a macro (read about [babel-plugin-macros](https://github.com/kentcdodds/babel-plugin-macros) if you are not familiar) for solving the "double declaration problem" in GraphQL queries. (mutations might be tackled some other day).

> **What is the "double declaration problem"?** Simply it is the bad developer experience of having to declare what you want to query in the GraphQL template string, and then again when you are using the data in your application. Ommissions are confusing to debug and overfetching due to stale queries is also a problem.

## Problem Statement

Here is a typical graphql query using [urql](https://codesandbox.io/s/p5n69p23x0) (taken [straight from urql's docs](https://github.com/FormidableLabs/urql#getting-started)):

```jsx
import { Connect, query } from "urql";

const QueryString = `
  query Movie($id: String) {
    movie(id: $id) {
      id,
      title,
      description,
      genres,
      poster {
        uri
      }
    }
  }

`;

const Movie = ({ id, onClose }) => (
  <div>
    <Connect
      query={query(QueryString, { id: id })}
      children={({ loaded, data }) => {
        return (
          <div className="modal">
            {loaded === false ? (
              <p>Loading</p>
            ) : (
              <div>
                <h2>{data.movie.title}</h2>
                <p>{data.movie.description}</p>
                <button onClick={onClose}>Close</button>
              </div>
            )}
          </div>
        );
      }}
    />
  </div>
);
```

you see how `title` and `description` are specified twice, while `poster` and `genre` aren't even used. 

## Example Input

Using `blade.macro`, you can now write:


```jsx
import { Connect, query } from "urql";
import { bladeQuery, BQL } from "blade.macro";

const Movie = ({ id, onClose }) => (
  <div>
    <Connect
      query={query(BQL, { id: id })} // `BQL` becomes a query string
      children={({ loaded, data }) => {
        const DATA = bladeQuery(data, {$id: "String"}) // initialize DATA as a blade, names Movie, passes params
        const movie = DATA.movie({ id: '$id' }) // `movie` is an alias, data.movie() has args. `movie` is also a blade now
        return (
          <div className="modal">
            {loaded === false ? (
              <p>Loading</p>
            ) : (
              <div>
                <h2>{movie.title}</h2>
                <p>{movie.description}</p>
                <button onClick={onClose}>Close</button>
              </div>
            )}
          </div>
        );
      }}
    />
  </div>
);
```

## Example Output

This transpiles to:

```jsx
import { Connect, query } from "urql";
const Movie = ({ id, onClose }) => (
  <div>
    <Connect
      query={query(`
        query Movie($id: String) {
          movie: movie(id: $id) {
            title,
            description
          }
        }
      `, { id: id })}
      children={({ loaded, data }) => {
        const movie = data.movie
        return (
          <div className="modal">
            {loaded === false ? (
              <p>Loading</p>
            ) : (
              <div>
                <h2>{movie.title}</h2>
                <p>{movie.description}</p>
                <button onClick={onClose}>Close</button>
              </div>
            )}
          </div>
        );
      }}
    />
  </div>
);
```

a key insight that makes this work:

- graphql data objects are never functions, so we can overload the "blade" as a function call to pass arguments to our graphql queries.
- aliases must be globally unique, just like variable declarations...


# Babel strategy

How we execute this code is a challenging task.

There are five big aspects to take care of:

- `makeBlade(name, args)` should tag the identifier it is assigned to (e.g. `blade`) as a special identifier. This is the start of the GraphQL query.
- when `blade(foo)` is called on `foo`, it tags `foo` as a "blade", which is a piece of a GraphQL query. any descendants of a blade are also blades
- blades can be called as functions to supply arguments to the blade
- assignments create aliases!
- wherever `blade` is referenced as a variable, the final GraphQL query is to be inserted

# What is a razor?

The job of a razor is:

- if a simple reference, to replace as a graphql query string
- if called, to make the assigned identifier into a blade. optionally passing along the query params.

One razor, many blades.

# What is a blade?

The job of a blade is:

- if the property is called as a function, add those arguments to the set
- if a property is accessed, add it to the graphql set
- if the property is assigned, make it an alias
  - search scope for more references, recursively

The name comes from "roller blade", also known as inline skates. The inspiration comes from a boss who described this as "inline GraphQL". But also it just sounds cool so there we go.

# Things to consider

accounting for as much of the [graphql spec](https://graphql.org/learn/queries/) as possible

- [ ] how to incorporate fragments?
- [x] [operation name](https://graphql.org/learn/queries/#operation-name)
- [x] nested function queries
- [x] aliases
- [x] query variables '$'
- [ ] directives
- [ ] [meta fields](https://graphql.org/learn/queries/#meta-fields)

---

# Alternative APIs considered

we could try a JSX version:

```jsx
import { Connect, query } from "urql";
import makeBlade from "blade.macro";

const MovieQuery = makeBlade() // query is named "MovieQuery"
const Movie = ({ id, onClose }) => (
  <div>
    <Connect
      query={query(MovieQuery, { id: id })}
      children={({ loaded, data }) => {
        return (
          <MovieQuery movie={id}>
            {data => (
              <div className="modal">
                {loaded === false ? (
                  <p>Loading</p>
                ) : (
                  <div>
                    <h2>{data.movie.title}</h2>
                    <p>{data.movie.description}</p>
                    <button onClick={onClose}>Close</button>
                  </div>
                )}
              </div>
            )}
          </MovieQuery>
        );
      }}
    />
  </div>
);
```
