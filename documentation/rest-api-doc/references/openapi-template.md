# OpenAPI 3.0 YAML Template

Use this as the base structure when generating OpenAPI output.

```yaml
openapi: 3.0.3

info:
  title: My API
  description: Short description of the API.
  version: 1.0.0
  contact:
    name: API Support
    email: support@example.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

security:
  - bearerAuth: []

tags:
  - name: Users
    description: User management endpoints
  - name: Items
    description: Item endpoints

paths:
  /users:
    get:
      tags: [Users]
      summary: List all users
      operationId: listUsers
      parameters:
        - name: page
          in: query
          required: false
          schema:
            type: integer
            default: 1
        - name: per_page
          in: query
          required: false
          schema:
            type: integer
            default: 20
      responses:
        "200":
          description: Paginated list of users
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/UserListResponse"
        "401":
          $ref: "#/components/responses/Unauthorized"

    post:
      tags: [Users]
      summary: Create a new user
      operationId: createUser
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateUserRequest"
      responses:
        "201":
          description: User created
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/User"
        "400":
          $ref: "#/components/responses/BadRequest"
        "401":
          $ref: "#/components/responses/Unauthorized"

  /users/{id}:
    get:
      tags: [Users]
      summary: Get a user by ID
      operationId: getUser
      parameters:
        - $ref: "#/components/parameters/IdParam"
      responses:
        "200":
          description: The requested user
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/User"
        "404":
          $ref: "#/components/responses/NotFound"

    put:
      tags: [Users]
      summary: Update a user
      operationId: updateUser
      parameters:
        - $ref: "#/components/parameters/IdParam"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/UpdateUserRequest"
      responses:
        "200":
          description: Updated user
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/User"
        "400":
          $ref: "#/components/responses/BadRequest"
        "404":
          $ref: "#/components/responses/NotFound"

    delete:
      tags: [Users]
      summary: Delete a user
      operationId: deleteUser
      parameters:
        - $ref: "#/components/parameters/IdParam"
      responses:
        "204":
          description: User deleted successfully
        "404":
          $ref: "#/components/responses/NotFound"

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  parameters:
    IdParam:
      name: id
      in: path
      required: true
      schema:
        type: string
      description: Resource identifier

  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          example: "abc123"
        name:
          type: string
          example: "Jane Doe"
        email:
          type: string
          format: email
          example: "jane@example.com"
        role:
          type: string
          enum: [admin, user]
          example: "user"
        createdAt:
          type: string
          format: date-time
          example: "2024-01-15T10:30:00Z"

    CreateUserRequest:
      type: object
      required: [name, email]
      properties:
        name:
          type: string
        email:
          type: string
          format: email
        role:
          type: string
          enum: [admin, user]
          default: "user"

    UpdateUserRequest:
      type: object
      properties:
        name:
          type: string
        email:
          type: string
          format: email
        role:
          type: string
          enum: [admin, user]

    UserListResponse:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: "#/components/schemas/User"
        meta:
          type: object
          properties:
            total:
              type: integer
            page:
              type: integer
            per_page:
              type: integer

    ErrorResponse:
      type: object
      properties:
        error:
          type: string
          example: "Resource not found"

  responses:
    BadRequest:
      description: Validation error
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    Unauthorized:
      description: Missing or invalid authentication
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
```

## Key Rules for OpenAPI Output

1. **Use `$ref` for anything reused** — error shapes, path params, security schemes.
2. **`operationId`** — camelCase, unique across the whole spec.
3. **Every path parameter** in a path string `{id}` must have a matching `parameters` entry with `in: path`.
4. **`security: []`** on an individual operation overrides the global default (use for public endpoints).
5. **`format`** hints matter — use `date-time`, `email`, `uuid`, `uri` where applicable.
6. **`example` values** should be realistic, not placeholder strings like `"string"`.