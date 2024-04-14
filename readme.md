# Example of relational data model being handled without relations in DynamoDB

Imagine we are designing a simple workout tracking app. Users would be able to log their workouts and see their progress over time. The app would have the following features:

1. Add a new workout type for a user (E.g. running, swimming, lifting, climbing, etc)
2. Log a new workout for a user
3. Query all workout types for a user
4. Query all workouts for a user
5. Query workouts for a user by type

## Relational data model
Let's assume that users are handled already either with a user table or a service like AWS Cognito. When seeing these requirements, a relational model feels very intuitive to keep data normalized.

### Tables

#### WorkoutType:
- id (PK)
- userId (FK)
- name
- description

#### Workout:
- id (PK)
- userId (FK)
- workoutTypeId (FK)
- duration
- calories
- timestamp

### Queries
1. Get all workout types for a user:  
`select * from WorkoutType where userId = :userId`
2. Get all workouts for a user:  
`select * from Workout where userId = :userId`
3. Get all workouts for a user of a specific type:  
`select * from Workout where userId = :userId and workoutTypeId = :workoutTypeId`

## DynamoDB NoSQL model
We will use a single table to store all the information. This table will be designed to handle multiple types of data entries (both workout types and workout results) using a composite sort key strategy.

### Table: Workouts
- Partition Key: UserID (String)
  - This key uniquely identifies each user.
- Sort Key: EntryType#Detail (String)
  - EntryType can be either the string "Type" or "Workout".
  - Detail varies:
    - For workout types, it is just the type name (e.g., "Swimming").
    - For workout results, it is a combination of the workout date and type (e.g., "2024-03-21#Swimming").
- Attributes:
  - Data: JSON or Map
    - Used to store attributes relevant to the entry, such as duration and calories for workouts, or description for workout types.

### Data Entry Examples
- Workout Types Entry (for User1, type Swimming)
  - Partition Key: "User1"
  - Sort Key: "Type#Swimming"
  - Data: { "Description": "Swimming Workout" }
- Workout Log Entry (for User1, swimming on March 21, 2024)
  - Partition Key: "User1"
  - Sort Key: "Workout#2024-03-21#Swimming"
  - Data: { "Duration": "30 minutes", "Calories": 300 }

### Access Patterns
1. Add a new workout type for a user: Insert a new item with the sort key starting with `Type#`.
2. Log a new workout for a user: Insert a new item with the sort key starting with `Workout#`, followed by the date and type.
3. Query all workout types for a user: Perform a query for entries where the sort key begins with `Type#`.
4. Query all workouts for a user: Perform a query for entries where the sort key begins with `Workout#`.
5. Query workouts for a user by type: Adjust the query to match the specific workout type in the sort key.

### Query Implementation Examples
#### Querying Workout Types for a User
```python
response = table.query(
  KeyConditionExpression='UserID = :userid and begins_with(EntryType#Detail, :typeprefix)',
  ExpressionAttributeValues={
    ':userid': 'User1',
    ':typeprefix': 'Type#'
  }
)
```

#### Querying Workouts for a User
```python
response = table.query(
    KeyConditionExpression='UserID = :userid and begins_with(EntryType#Detail, :workoutprefix)',
    ExpressionAttributeValues={
        ':userid': 'User1',
        ':workoutprefix': 'Workout#'
    }
)
```
