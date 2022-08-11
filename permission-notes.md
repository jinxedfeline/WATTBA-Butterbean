# Bot Permission Management Ideas

The aim is to have a generalized way of giving permissions to users, based on either their user id or their role.

For the user id, we'll use the more intuitive name plus discriminator format, i.e. SomeUser#1234, as is returned
by a user object in string context (and already used as such in the bot previously), rather than the immutable
id, which is numeric and quite long.

For roles, we'll go simply by their names, as ids are similarly unwieldy. We could make things more complicated
later by adding another option to give permissions based not on the roles themselves, but the permissions
granted to a user by their roles. That doesn't seem necessary at this point.

### I would propose the following table format:

Each row contains either a user name or a role name, and then one boolean column per associated permission.
That way, we can easily query a user's roles and individual permissions to see if any of them match a required
permission, without needing a lot in the way of extra complications. If we want to get more intricate, we could
get into having more nuanced permissions, for example having user-specific permissions/lack thereof override
role-wide ones, but again, simple should suffice for now.

The table could easily be used not just to provide permissions to users, but also to flag certain roles for
some of the bot functions, for example to mark roles that users are allowed to set on themselves (like pronouns
and voluntary group memberships), and to mark roles that someone with the right permissions can ask the bot to
apply to other users.

This is a proposed way the table could look like. It only looks fancy because of the column/table constraints
that keep us from adding data that makes no sense - we check each row to have exactly one of user or role,
and we check on a flag that only makes sense on roles to ensure that it's set alongside a role.

```sql
CREATE TABLE "permissions" (

  "user" TEXT UNIQUE, -- a user name, a la "SomeUser#1234"; it makes sense to have only one entry per user
  "role" TEXT UNIQUE, -- a role name, a la "she/her"; it makes sense to have only one entry per role

  -- we use INTEGER because sqlite does not know booleans and represents them with 1 and 0
  "manage_memes" INTEGER NOT NULL DEFAULT 0, -- this user/role is allowed to add/remove memes
  "assign_roles" INTEGER NOT NULL DEFAULT 0, -- this user/role is allowed to add/remove roles on users
  "bot_admin"    INTEGER NOT NULL DEFAULT 0, -- this user/role has bot admin permissions

  -- this role can be set by users on themselves
  "is_user_settable" INTEGER NOT NULL DEFAULT 0
  -- ensure this is only set on roles
  CONSTRAINT "is_user_settable requires a role" CHECK(
    (("is_user_settable" == 1) AND ("role" IS NOT NULL)) OR
    ("is_user_settable" == 0)
  ),

  -- this role can be set on users by someone with the assign_roles permission
  -- note that this will not override permissions on the discord side, the bot is bound by which roles
  --   it has permission to assign there
  "is_assignable" INTEGER NOT NULL DEFAULT 0
  -- ensure this is only set on roles
  CONSTRAINT "is_assignable requires a role" CHECK(
    (("is_assignable" == 1) AND ("role" IS NOT NULL)) OR
    ("is_assignable" == 0)
  ),
    
  -- make sure we have a clean divide between rows for a specific user's permissions and rows for role
  --  permissions    
  CONSTRAINT "provide only one of: user or role" CHECK(
    (("user" IS NOT NULL) AND ("role" IS NULL)) OR
    (("user" IS NULL) AND ("role" IS NOT NULL))
  )

)
```