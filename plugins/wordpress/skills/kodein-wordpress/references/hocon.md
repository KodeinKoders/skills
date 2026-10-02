# Reading the credentials conf file

The file is [HOCON](https://github.com/lightbend/config/blob/main/HOCON.md).
You need the resolved values of `auth.username` and `auth.password`.
All of these are equivalent:

```hocon
auth {
    username = "editor"
    password = "secret"
}
```

```hocon
auth.username = editor
auth.password = secret
```

```hocon
auth { username: editor, password: secret }
```

```hocon
auth = { "username": "editor", "password": "secret" }
```

```hocon
auth { username = editor }
auth { password = secret }   # objects defined twice are merged
```

## Things to handle

- **Comments**: `#` and `//` start a comment outside quoted strings.
- **Separators**: `=`, `:`, or none before `{`. Commas or newlines separate fields.
- **Quoting**: unquoted values end at the end of the line (or `,` `}`) and are trimmed; quoted values keep spaces, and may contain `\"`, `\\`, `\n`, `\uXXXX`. `"""triple quoted"""` is literal.
- **Later wins**: if a key is defined several times, the last definition wins (objects merge).
- **Substitutions**: `password = ${MY_ENV_VAR}` reads the environment variable; `${?MY_ENV_VAR}` is optional and leaves the key unset when the variable does not exist; `${auth.other}` refers to another key of the file.
- **Concatenation**: `username = ${USER}"@wp-example.com"`.
- **Includes**: `include "other.conf"` — resolve relative to the file's directory.
- **Other keys** may be present (`url`, `site`, …); ignore the ones you do not need, never assume they are absent.

If a real HOCON parser is conveniently available (e.g. `hocon` / Typesafe Config via JVM, `pyhocon` via Python), you may use it, printing only whether the keys resolved, never their values.
Otherwise, read the file and resolve the two keys by hand following the rules above.
