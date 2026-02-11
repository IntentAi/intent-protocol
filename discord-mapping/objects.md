# Discord API Mapping - Objects

Intent object → Discord object shape mapping.

## Server (Guild)

```json
{
 "id": "snowflake",
 "name": "string",
 "owner_id": "snowflake",
 "icon": "string | null",
 "roles": [/* Role objects */],
 "channels": [/* Channel objects */]
}
```

Maps directly to Discord's Guild object.

 Full object mapping in development
