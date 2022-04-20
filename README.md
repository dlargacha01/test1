# Project 2

## Group Members:
Diego Largacha Urrutia: djl2188 <br>
Jordhy Gonzalez: jg4208

## Name of PostgreSQL Account
UNI: djl2188

## Modifications to Schema
We chose to implement an array structure into our players table, two composite
types, one for our non-keepers and one for our goalkeepersi, and lastly a
trigger to check for our >=1 constraint on teams that we have wanted to implement
since the beginning. Each is discussed more thoroughly below.


### Insertion of Array into players schema:
In order to represent the fact that one player can play for multiple teams over
the span of their career, we decided to implement an array called teams that
contains all the unique teams that each player has palyed for. They are organized
in order of first appearance and based on real-life data for each player in our
existing database. In order to do this, we just had to add one attribute to our
players entity by adding an additional piece of data in our table schema, shown
below: <br>

```
CREATE TABLE players(
        player_id int NOT NULL,
        club_id int,
        since date NOT NULL,
        first_name varchar(20) NOT NULL,
        surname varchar(20) NOT NULL,
        nationality varchar(20) NOT NULL,
        age int NOT NULL CHECK (age>=0),
        dob date NOT NULL,
        position varchar(2) NOT NULL,
        salary numeric(12,2) NOT NULL CHECK (salary>=0),
        price_bought numeric(12,2) CHECK (price_bought>=0),
        matches_played int NOT NULL CHECK (matches_played>=0),
        passes_cmp_p real NOT NULL CHECK(passes_cmp_p>=0 AND passes_cmp_p<=100),
        fouls int NOT NULL CHECK (fouls>=0),
--->    teams text[],
        PRIMARY KEY (player_id),
        FOREIGN KEY (club_id) REFERENCES clubs ON DELETE SET NULL
);
```
<br>

As mentioned, this allows us to perform searches on the players not only by their
current team, but also past teams, something which is very useful for our cases.

### Composite Types in `non_keepers` and `goalkeepers`:

The database that we are using goes into much more depth than we were initiallity
going into, with multiple different stats/perspectives for things like goals scored
or goals conceded. We felt that composite types would be a great way to show how
some of these stats all relate to the same thing, such as goals, but are different
stats in their own right. Thus, we added two new goal statistics under the `goal_types`
composite type, specifically goals not scored as a penalty and the expected non-penalty
goals scored per 90 minutes. As for goalkeepers, we made another composite type separating
between goals conceded. We have the number of goals scored as penalties, corners, free kick,
own goals, and the expected number of goals allowed per shot on target. We felt including
these expected statistics would be the most interesting as they are an even better way to
compare two different players. We can see the composite schemas below: <br>

```
CREATE TYPE goal_types AS (
        total int,
        non_pen int,
        ex_non_pen_per90 real
);


CREATE TYPE conceded_types AS (
        total int,
        pen_allowed int,
        free_kicks int,
        corner_kicks int,
        own_goals_against int,
        ex_conceded_per_SOT real
);
```
<br>

We then included these in our old table schemas as follows: <br>

```
CREATE TABLE goalkeepers(
        player_id int NOT NULL,
        save_p numeric(3,1) CHECK (save_p >=0 AND save_p<=100),
--->    conceded conceded_types,
        PRIMARY KEY (player_id),
        FOREIGN KEY (player_id) REFERENCES players ON DELETE CASCADE
);


CREATE TABLE non_keepers(
        player_id int NOT NULL,
--->    goals goal_types NOT NULL,
        assists int NOT NULL CHECK (assists>=0),
        p_on_target numeric(3,1) CHECK (p_on_target>=0 AND p_on_target<=100),
        PRIMARY KEY (player_id),
        FOREIGN KEY (player_id) REFERENCES players ON DELETE CASCADE
);
```
<br>

Overall, the inclusion of these composites allow us to include more detailed stats
for each player without making our schema more confusing as the composites
allow us to clearly show that all of the stats within the composite are related
to each other. <br>

### Triggers for >=1 Assertion:
Ever since the beggining of our project, we have been wanting to implement the >=1 participation
constraint for employees and players on their respective relationship with teams. In other words,
We want to make it so that no club can have no players or no employees. However,
as we mentioned in our project 1 part 2, we haven't been able to add those into our database until
we could use triggers. We have thus now added this along with additional functionality.
After attempting to delete a player or employee, we check to see if there are any clubs
which have no employees or no players, depending on which was deleted. If the club which has just
experienced the delete has no employees or no players, we delete the club as well. In the case
where we delete all the players of a club, and due to our IF DELETE CASCADE functionality in employees,
management, coaches, and managed, deleting the club will result in the deletion of all employees who
were working at the club. This exists because we don't want employees to exist in the database who
aren't working at a club. In the case where we delete all employees from a club, the players associated
with the club are not deleted because we wouldn't want to lose all the stats from these players.<br>


Here are the two triggers we have created:<br>

```
CREATE OR REPLACE FUNCTION nonemptyemps()
  RETURNS TRIGGER
  LANGUAGE PLPGSQL
  AS
$$
DECLARE
    data record;
    exec text;
BEGIN
        FOR data in (SELECT club_id FROM clubs WHERE clubs.club_id NOT IN (SELECT clubs.club_id FROM clubs NATURAL JOIN employees))    
    LOOP
        exec := 'DELETE FROM clubs * WHERE club_id = '||data.club_id;
        EXECUTE exec;
    END LOOP;

        RETURN NEW;
END;
$$;


CREATE TRIGGER nonemptyemp
  AFTER DELETE
  ON employees
  EXECUTE PROCEDURE nonemptyemps();


CREATE OR REPLACE FUNCTION nonemptyteam()
  RETURNS TRIGGER
  LANGUAGE PLPGSQL
  AS
$$
DECLARE
    data record;
    exec text;
BEGIN
        FOR data in (SELECT club_id FROM clubs WHERE clubs.club_id NOT IN (SELECT clubs.club_id FROM clubs NATURAL JOIN players))
    LOOP
        exec := 'DELETE FROM clubs * WHERE club_id = '||data.club_id;
        EXECUTE exec;
    END LOOP;

        RETURN NEW;
END;
$$;


CREATE TRIGGER nonemptyteams
  AFTER DELETE
  ON players
  EXECUTE PROCEDURE nonemptyteam();
```
<br>


## Interesting Queries
Consider the following interesting queries:
<br>
### Query on Arrays in players
```
SELECT p.first_name, p.surname, c1.name, (c1.points-c2.points) AS point_difference
FROM players p, clubs c1, clubs c2
WHERE c1.club_id = p.club_id AND 'Arsenal' = ANY (p.teams) AND c1.name != 'Arsenal' AND c2.name = 'Arsenal';
```
<br>

Output: <br>

```
 first_name |  surname  |    name     | point_difference
------------+-----------+-------------+------------------
 Lukasz     | Fabianski | West Ham    |                0
 Emiliano   | Martinez  | Aston Villa |              -15
(2 rows)
```
<br>

In this query, we print the full name, current team, and difference in points between
their current team and Arsenal for all players who used to play in Arsenal but no
longer do. In essence, it shows us whether ex-Arsenal players are doing better off
in points in their current teams, which would of course be useful in a soccer database
and only possible thanks to the array that we implemented in players.<br>


### Query on Composite Types
```
SELECT p.first_name, p.surname, (g.conceded).ex_conceded_per_SOT AS Expected_Conceded_per_Shot_on_Target, c.name AS club
FROM players p, goalkeepers g, clubs c
WHERE p.club_id = c.club_id AND p.player_id = g.player_id AND (g.conceded).own_goals_against = 0
      AND (g.conceded).ex_conceded_per_SOT < ALL (SELECT AVG((g2.conceded).ex_conceded_per_SOT)
                                                  FROM goalkeepers g2)
ORDER BY (g.conceded).ex_conceded_per_SOT ASC;
```
<br>

Output: <br>

```
 first_name |      surname      | expected_conceded_per_shot_on_target |      club
------------+-------------------+--------------------------------------+-----------------
 Aaron      | Ramsdale          |                                 0.22 | Arsenal
 Ederson    | Santana de Moraes |                                 0.26 | Manchester City
 George     | Jenkins           |                                 0.26 | Temp Team
(3 rows)
```
<br>


In this query, we print the full names, club, and expected goal conceded per shot on traget for all
goalkeepers who are below the average expected goal allowed per shot on target. This makes use of
the new composite types used in goalkeepers and allows us to see who the best goalkeepers are in
expectation, which is of course useful for a database surrounding game statistics. <br>

### Query on triggers
```
DELETE FROM players * WHERE club_id = 100
```
<br>
Or alternatively, 
<br>
```
DELETE FROM employees * WHERE club_id = 100
```
<br>

Before running either of these queries, a club with `club_id=100` exists and contains both players
and employees. <br>

```
SELECT c.club_id, c.name,
        (SELECT COUNT(*) FROM employees e WHERE e.club_id = 100) AS num_employees,
        (SELECT COUNT(*) FROM players p WHERE p.club_id = 100) AS num_players
FROM clubs c
WHERE c.club_id = 100;

DELETE FROM players WHERE club_id = 100;

SELECT count(c.club_id) AS num_teams,
        (SELECT COUNT(*) FROM employees e WHERE e.club_id = 100) AS num_employees,
        (SELECT COUNT(*) FROM players p WHERE p.club_id = 100) AS num_players
FROM clubs c
WHERE c.club_id = 100;
```
<br>

Will output: <br>

```
 club_id |   name    | num_employees | num_players
---------+-----------+---------------+-------------
     100 | Temp Team |             3 |           2
(1 row)

DELETE 2

 num_teams | num_employees | num_players
-----------+---------------+-------------
         0 |             0 |           0
(1 row)
```
<br>

As we can see, in the example of deleting all the players from Temp Team, we can see that it will delete the
Team and also delete all employees with the associated `club_id`. In the case of deleting all the employees,
the players associated with the team will have NULL as their `club_id`. This is in line with our wish of not
having any empty teams and not having any employees that don't work at any teams.

