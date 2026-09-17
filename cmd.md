# Perform Predictive Data Analysis in BigQuery: Challenge Lab || **GSP374**

**Command:**

```bash
# ==========================================
# 1. CONFIGURATION (VERIFY NAMES IF NEEDED)
# ==========================================
export PROJECT_ID=$(gcloud config get-value project)
export DATASET_NAME=""
export MODEL_NAME=""
export FUNCTION_DIST_NAME=""
export FUNCTION_ANGLE_NAME=""

# ==========================================
# TASK 1: DATA INGESTION
# ==========================================
bq load --source_format=NEWLINE_DELIMITED_JSON --autodetect ${DATASET_NAME}.events gs://spls/bq-soccer-analytics/events.json
bq load --source_format=CSV --autodetect ${DATASET_NAME}.tags2name gs://spls/bq-soccer-analytics/tags2name.csv

# ==========================================
# TASK 2: ANALYZE PENALTY KICK SUCCESS RATE
# ==========================================
bq query --use_legacy_sql=false \
"SELECT
  playerId,
  (Players.firstName || ' ' || Players.lastName) AS playerName,
  COUNT(id) AS numPKAtt,
  SUM(IF(101 IN UNNEST(tags.id), 1, 0)) AS numPKGoals,
  SAFE_DIVIDE(SUM(IF(101 IN UNNEST(tags.id), 1, 0)), COUNT(id)) AS PKSuccessRate
FROM
  \`${DATASET_NAME}.events\` Events
LEFT JOIN
  \`${DATASET_NAME}.players\` Players ON Events.playerId = Players.wyId
WHERE
  eventName = 'Free Kick' AND subEventName = 'Penalty'
GROUP BY
  playerId, playerName
HAVING
  numPKAtt >= 5
ORDER BY
  PKSuccessRate DESC, numPKAtt DESC"

# ==========================================
# TASK 3: ANALYZE SHOT DISTANCE
# ==========================================
bq query --use_legacy_sql=false \
"WITH Shots AS (
  SELECT
    *,
    (101 IN UNNEST(tags.id)) AS isGoal,
    SQRT(
      POW((100 - positions[ORDINAL(1)].x) * 105/100, 2) +
      POW((50 - positions[ORDINAL(1)].y) * 68/100, 2)
    ) AS shotDistance
  FROM
    \`${DATASET_NAME}.events\`
  WHERE
    eventName = 'Shot' OR
    (eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty'))
)
SELECT
  ROUND(shotDistance, 0) AS ShotDistRound0,
  COUNT(*) AS numShots,
  SUM(IF(isGoal, 1, 0)) AS numGoals,
  AVG(IF(isGoal, 1, 0)) AS goalPct
FROM
  Shots
WHERE
  shotDistance <= 50
GROUP BY
  ShotDistRound0
ORDER BY
  ShotDistRound0"

# ==========================================
# TASK 4: USER DEFINED FUNCTIONS & MODEL
# ==========================================
# 4a. Create Shot Distance UDF
bq query --use_legacy_sql=false \
"CREATE OR REPLACE FUNCTION \`${FUNCTION_DIST_NAME}\`(x INT64, y INT64)
RETURNS FLOAT64 AS (
  SQRT(
    POW((100 - x) * 105/100, 2) +
    POW((50 - y) * 68/100, 2)
  )
);"

# 4b. Create Shot Angle UDF
bq query --use_legacy_sql=false \
"CREATE OR REPLACE FUNCTION \`${FUNCTION_ANGLE_NAME}\`(x INT64, y INT64)
RETURNS FLOAT64 AS (
  SAFE.ACOS(
    SAFE_DIVIDE(
      (
        (POW(105 - (x * 105/100), 2) + POW(34 + (7.32/2) - (y * 68/100), 2)) +
        (POW(105 - (x * 105/100), 2) + POW(34 - (7.32/2) - (y * 68/100), 2)) -
        POW(7.32, 2)
      ),
      (2 *
        SQRT(POW(105 - (x * 105/100), 2) + POW(34 + 7.32/2 - (y * 68/100), 2)) *
        SQRT(POW(105 - (x * 105/100), 2) + POW(34 - 7.32/2 - (y * 68/100), 2))
      )
    )
  ) * 180 / ACOS(-1)
);"

# 4c. Train Expected Goals Logistic Regression Model
bq query --use_legacy_sql=false \
"CREATE OR REPLACE MODEL \`${MODEL_NAME}\`
OPTIONS(
  model_type='logistic_reg',
  input_label_cols=['isGoal']
) AS
SELECT
  (101 IN UNNEST(tags.id)) AS isGoal,
  subEventName AS shotType,
  \`${FUNCTION_DIST_NAME}\`(positions[ORDINAL(1)].x, positions[ORDINAL(1)].y) AS shotDistance,
  \`${FUNCTION_ANGLE_NAME}\`(positions[ORDINAL(1)].x, positions[ORDINAL(1)].y) AS shotAngle
FROM
  \`${DATASET_NAME}.events\` Events
LEFT JOIN
  \`${DATASET_NAME}.competitions\` Competitions ON Events.competitionId = Competitions.wyId
WHERE
  Competitions.name != 'World Cup' AND
  (eventName = 'Shot' OR (eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty')))"

# ==========================================
# TASK 5: MAKE PREDICTIONS ON NEW DATA
# ==========================================
bq query --use_legacy_sql=false \
"SELECT
  *
FROM
  ML.PREDICT(
    MODEL \`${MODEL_NAME}\`,
    (
      SELECT
        (101 IN UNNEST(tags.id)) AS isGoal,
        subEventName AS shotType,
        \`${FUNCTION_DIST_NAME}\`(positions[ORDINAL(1)].x, positions[ORDINAL(1)].y) AS shotDistance,
        \`${FUNCTION_ANGLE_NAME}\`(positions[ORDINAL(1)].x, positions[ORDINAL(1)].y) AS shotAngle
      FROM
        \`${DATASET_NAME}.events\` Events
      LEFT JOIN
        \`${DATASET_NAME}.competitions\` Competitions ON Events.competitionId = Competitions.wyId
      WHERE
        Competitions.name = 'World Cup' AND
        (eventName = 'Shot' OR (eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty')))
    )
  )"

echo "==> All lab tasks finished successfully!"
