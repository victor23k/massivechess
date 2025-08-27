# Massive Chess

## Front-end / UI

- Chess board
- Send button, go to `/games` to see results or progress
- On send, update `localStorage` with the game `uuid`.

### `/index`

Board with a random game to play.

### `/games`

Shows all games played. Finished games show results, number of unique players and how many times you played. We could
rank players based on move score.

### `localStorage`

The db for the front-end will be `localStorage`. For now we don't want to identify users.
Things to store:

- `games_played`: array of games ids. Represents all the games played by the user, finished or ongoing.
- `user_id`: self explanatory.

## Back-end

- Generate user id to identify players in a game.

### DB

#### `games`

 Column        |  Type     | Collation | Nullable | Default
---------------+-----------+-----------+----------+---------
 id            | uuid      |           | not null | 
 user_to_move  | uuid      |           |          | null
 finished      | boolean   |           | not null | false
 inserted_at   | timestamp |           | not null | 
 updated_at    | timestamp |           | not null |

 foreign_key `user_to_move` references `users`.

#### `users`

 Column        |  Type     | Collation | Nullable | Default
---------------+-----------+-----------+----------+---------
 id            | uuid      |           | not null | 
 inserted_at   | timestamp |           | not null | 
 updated_at    | timestamp |           | not null | 

#### `moves`

 Column        |  Type     | Collation | Nullable | Default
---------------+-----------+-----------+----------+---------
 id            | uuid      |           | not null | 
 game          | uuid      |           | not null | 
 user          | uuid      |           | not null | 
 score         | integer   |           | not null | 
 inserted_at   | timestamp |           | not null | 
 updated_at    | timestamp |           | not null | 

 index `user`.
 foreign_key `game` references `games`.
 foreign_key `user` references `users`.

### Logic

#### Pick a game to show

Choose an available game (more on this below) or a new one. If there are available games, start a new one has a 5%
chance.

#### Locking games

When a game is shown to a user, it must be locked first. This lock is `games.user_to_move`.
When the user submits their move and the move is processed, the game is unlocked, setting the `user_to_move` to null.

We still probably need more locking at the application layer, because if two players request a game at the same time,
they could get the same result set of available games.

The application layer lock: 
- starts when a game is assigned to a user,
- releases when all result sets with that game to null get assigned other game.

So we could have some sort of ref counts.

##### Ref counting locked games

`PlayGame` gets a result set with `LIMIT 5`. It adds one observer to all those 5 games and tries to lock one.
If it gets a lock, commits that game to db and sends the response with the game to play.
If it does not get a lock:

- gets the next 5 games and repeat process, or
- starts a new game with the `user_to_move` already set.

When a game loses all refcounts, drop the game from the lock cache.

#### Unlocking stale games

To prevent stale games, a process cleans up locked games **at the DB layer** every 24 hours.

#### Show past games

Finds all `moves` for `user` grouped by `game`.
Then it needs to reconstruct games, finished and ongoing by gathering all `moves` by `game`.

Show moves made by the user.
