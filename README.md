# blade.macro

this is a macro for solving the "double declaration problem" in GraphQL queries. (mutations might be tackled some other day).

> **What is the "double declaration problem"?** Simply it is the bad developer experience of having to declare what you want to query in the GraphQL template string, and then again when you are using the data in your application. Ommissions are confusing to debug and overfetching due to stale queries is also a problem.

## Example

Here is a typical graphql query using [urql](https://codesandbox.io/s/p5n69p23x0):

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

`blade.macro` would turn the query into:


```jsx
import { Connect, query } from "urql";
import makeBlade from "blade.macro";

const blade = makeBlade('Movie', {id: "String"}) //optional naming
const Movie = ({ id, onClose }) => (
  <div>
    <Connect
      query={query(blade, { id: id })} // `blade` becomes a query string
      children={({ loaded, data }) => {
        blade(data) // gets compiled away
        const movie = data.movie({ id }) // this also becomes a blade
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

This compiles to:

```jsx
import { Connect, query } from "urql";
const Movie = ({ id, onClose }) => (
  <div>
    <Connect
      query={query(`
        query Movie($id: String) {
          movie(id: $id) {
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
