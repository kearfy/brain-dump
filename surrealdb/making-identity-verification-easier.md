

# Making identity verification easier

**By:** Micha de Vries 

**Platform:** SurrealDB, Beta 8

--- 
<br>




## The Problem

It is very easy to set up scope authentication for SurrealDB, unfortunately it is not as easy to validate those users (email, phone, totp, etc). You could then ofcourse use an external authentication provider to create a token for you, but it's nice to have all of this functionality inclusive in one platform. This paper is an attempt to find a solution to that problem.


<br>

## Current situation

Let's say we have a table where we store our users:

```sql
DEFINE TABLE user SCHEMAFULL
    PERMISSIONS
        FOR select FULL
        FOR update, delete WHERE $scope = 'user' && id = $auth.id;

DEFINE FIELD name ON TABLE user TYPE string ASSERT $value != NONE;
DEFINE FIELD email ON TABLE user TYPE string ASSERT is::email($value);
DEFINE FIELD password ON TABLE user TYPE string ASSERT $value != NONE;
```

Now, these users will have to authenticate with a scope to be able to update or delete their account:

```sql
DEFINE SCOPE user 
    SIGNIN ( SELECT * FROM user WHERE email = $email AND crypto::argon2::compare(password, $password) )
;
```

That is easily done, we now have a working signin system! Difficulty arises when we start to talk about user signup, or password resets. At that stage we will have to verify that they actually are the owner of that email address. It's possible, but it currently takes multiple tables and events to get this done securely. Procedures that should soon be a think will already majorly simplify this, but it's still too complicated for what it is and _very_ prone to bugs.

Don't get me wrong, this is not about forming surrealdb into an email server or an sms sending service. This paper will rather discuss potential improvements to it's query language that will likely also help to simplify other processes.


<br>


## Initial thoughts

Okay, let's collect some thoughts. I'm writing this down as I'm thinking of solutions, basically a dump of my brain, so bear with me.

First dillema that comes to mind is if the solution should be a statefull or stateless process. Meaning, will we store data in the database to later verify _something_, or will we use something like JWT tokens to verify state. Nice thing about a statefull process is that you can see exactly where in the process a user is, and it might be easier to help them out if they're stuck. For a stateless process, well, I like a stateless process lol. Guess I'll try to cover both, maybe it's even possible to create a process that covers both :D

I'd personally like this to work as some of plugin, but it has to be something you can define in surrealql. Not like the php mysql plugin or something.


<br>


## Pling! Idea #1

I've been using the word process an awefull lot, because that is exactly the thing it is. Verifying your email is a process where you go from step A (unverified), to step B (verified). What if you can define some sort of process, with state, that table definitions can hook into?

### What would a process... be?

So this is totally unrelated, but do you know these sort of blocks of code in blockchain, etherium right? So what if a process is a special table of some sort, that can store state just the way that tables work in surrealdb?

```sql
DEFINE PROCESS verify_email 
    ENTRY {
        -- you can get & set state here, while the outside cannot. 
        -- Only with actions they invoke.
    }

    EXIT {
        -- Has a return value that gets set as the final state.
        -- How would you pass arguments to this "action" though?
    }
;
```

### Storing the actual state

Just define field for the process! The `ASSERT`'s and `VALUE`'s are only valid for the initial input for the process. The actions within the process can do whatever they want, as long as the type for the field is correct. 

The value for the fields needs to stay aligned with the types, because external libraries might rely on it when querying data.

The assert & value is also not nessacary anymore, as the state can only be altered by actions in processes which are pre-defined.

```sql
DEFINE FIELD user TYPE record(user) ASSERT $value != NONE;
DEFINE FIELD email TYPE string ASSERT is::email($value);
DEFINE FIELD secret TYPE string;
```

### Passing state to actions in a process

This section does not get into executing an action just yet, rather how you can define arguments for an action in a process.

For the entry this is pretty obvious, a user has to input accordingly to the defined fields.

Maybe we can follow some JS sort of syntax?

```sql
EXIT ($secret: string, $other?: number) {

}
```

then `$secret` is required, and `$other` is optional? Hmm, maybe. I'll get back to it later. For now this works.