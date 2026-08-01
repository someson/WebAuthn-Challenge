# WebAuthn Series

## WebAuthn 1
Description:
```markdown
Passwords are SO OLD SCHOOL, passwordless authentication is the way of the future! [WebAuthentication](https://w3c.github.io/webauthn/) is a pretty cool spec, so I wrote my website to use it. However, I'm not much of a front-end guy, so all I've left you with is an API. Can you log in?

[server.py]
```

## WebAuthn 2
Description:
```markdown
Alright, so you now know how to use WebAuthn. [But can you hack it](https://essay.utwente.nl/98532/1/Chen_MA_EEMCS.pdf)?? If you can log in as admin, you'll get a flag.

This challenge uses the same files and remote server as WebAuthn 1.

```

# Hosting
This challenge should be a Docker container that runs `server.py` on port 1337. All the proper files are included in here. The command to build the docker container is (when located inside of this directory):

```bash
docker compose up -d --build webauthn
```

To stop the challenge:
```bash
docker compose down
```
