# 💾 README: Custom SQL Database Agent (ReAct)

## 1.  Purpose

This project implements a single-purpose, read-only **SQL Database Agent** built entirely from scratch. The agent interprets natural language queries against a **Film/TV SQLite database** by strictly following the **ReAct (Reasoning + Acting)** loop.

The goal is to demonstrate structured problem-solving: the agent must use its tool system to explore the database schema, safely formulate complex SQL queries (including joins and filtering), and provide a final answer, all while generating an auditable, step-by-step trace.

### Database Schema (Film/TV)

| Table | Columns | Purpose |
| :--- | :--- | :--- |
| **movies** | `movie_id`, `title`, `release_year`, `genre`, `rating` | Core movie information and ratings. |
| **actors** | `actor_id`, `name` | Actor details. |
| **movie\_actors** | `movie_id`, `actor_id`, `role` | Junction table for the many-to-many relationship between movies and actors. |

***

## 2.  How to Run (Colab Environment)

### Setup
1.  **Dependencies:** This project requires the `sqlite3` standard library and the `google-genai` client library.
2.  **API Key:** The Gemini API key must be configured in an initial Python cell (e.g., using `os.environ` or `genai.configure`) before the agent is initialized.
3.  **Execution:** The entire source code (DB setup, tool definitions, and agent logic) must be executed in preceding cells. The agent is then run by calling the `query` function directly in a new cell.

### Sample Execution Command

Execute the following in a Colab code cell to run the agent:


query("List the titles of all movies that star Leonardo DiCaprio and were released after the year 2000.")

## 3\.  Tool System Description

The agent interacts with the environment exclusively through three predefined, custom-implemented tools. All actions must adhere to the format: **`Action: tool_name{"arg1": "value"}`**.

| Tool Name | Arguments (JSON) | Returns (Observation) | Safety Constraint |
| :--- | :--- | :--- | :--- |
| **`list_tables`** | `{}` | A list of all table names (e.g., `['movies', 'actors']`). | N/A |
| **`describe_table`** | `{"table_name": "TEXT"}` | The table schema: a list of `(column_name, data_type)` tuples. | Checks for table existence. |
| **`query_database`** | `{"query": "SQL_SELECT_QUERY"}` | A dictionary containing results: `{"columns": [names], "rows": [data]}`. | Enforces **read-only `SELECT`** and appends a `LIMIT 100` if not present. |

-----

## 4\. Example Run with Trace

The following trace demonstrates the agent's multi-step reasoning to answer a question requiring schema inspection and a complex three-table join.

The agent successfully determines that it needs to describe all three tables before synthesizing the correct SQL query, which joins `movies`, `movie_actors`, and `actors` tables.

### Query

`query("List the titles of all movies that star Leonardo DiCaprio and were released after the year 2000.")`

### Execution Trace

```
Thought: I need to find movie titles, actor information, and release dates. I should start by listing the tables to see which tables might contain this information.
Action: list_tables{}
PAUSE
Observation: ['movies', 'actors', 'movie_actors']

Thought: I have three tables: 'movies', 'actors', and 'movie_actors'. I need to verify their schemas to correctly structure the query that joins all three. First, 'actors'.
Action: describe_table{"table_name": "actors"}
PAUSE
Observation: [('actor_id', 'INTEGER'), ('name', 'TEXT')]

Thought: The 'actors' table has 'actor_id' and 'name'. Next, check the 'movies' table for title and year.
Action: describe_table{"table_name": "movies"}
PAUSE
Observation: [('movie_id', 'INTEGER'), ('title', 'TEXT'), ('release_year', 'INTEGER'), ('genre', 'TEXT'), ('rating', 'REAL')]

Thought: The 'movies' table has 'movie_id', 'title', and 'release_year'. Finally, check the linking table 'movie_actors'.
Action: describe_table{"table_name": "movie_actors"}
PAUSE
Observation: [('movie_id', 'INTEGER'), ('actor_id', 'INTEGER'), ('role', 'TEXT')]

Thought: I have confirmed all schemas. I will construct the final query to join all three tables, filter by actor name 'Leonardo DiCaprio' and release year > 2000, and select the movie titles.
Action: query_database{"query": "SELECT T1.title FROM movies AS T1 INNER JOIN movie_actors AS T2 ON T1.movie_id = T2.movie_id INNER JOIN actors AS T3 ON T2.actor_id = T3.actor_id WHERE T3.name = 'Leonardo DiCaprio' AND T1.release_year > 2000;"}
PAUSE
Observation: {'columns': ['title'], 'rows': [('Inception',), ('The Dark Knight',)]}

Thought: The query returned the titles: Inception and The Dark Knight. This is the final answer.
FINAL ANSWER: The titles of movies starring Leonardo DiCaprio released after 2000 are: Inception, The Dark Knight.
```

