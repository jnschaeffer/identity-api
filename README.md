# dmv - please proceed to the next kiosk

dmv is an OAuth service that supports the following grant types:

* [RFC 8693][rfc8693]: Token Exchange

[rfc8693]: https://www.rfc-editor.org/rfc/rfc8693.html

## Usage

dmv is a Go service. To build it, you can either use `make build` to build a Go binary or `make docker-up` to start the Docker Compose service.

The `docker-up` Makefile target will auto-generate a private key and mount it in the container for testing purposes. Note that this is not recommended for actual production use, and is merely a handy feature to allow developers to test.

### Exchanging tokens

To perform a token exchange, grab a JWT from somewhere and add its issuer and JWKS URI to the `oauth.subjectTokenIssuers` section of your config file. Then, try running the following:

```
$ read -s -p 'Enter your token: ' AUTH_TOKEN && echo
$ curl --user my-client:foobar -XPOST -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange&subject_token=$AUTH_TOKEN&subject_token_type=urn:ietf:params:oauth:token-type:jwt" http://localhost:8000/token | jq
```

This sends an RFC 8693 token exchange request to dmv, which then will validate the given subject token against the configured subject token issuers. If the token is valid, you will receive a response like so (access token truncated for brevity):

```
{
  "access_token": "eyJ..VwM",
  "expires_in": 100,
  "token_type": "urn:ietf:params:oauth:token-type:jwt"
}
```

To examine the payload of the access token JWT itself, you can use [jq][jq] to decode the payload:

```
$ echo "$ACCESS_TOKEN" | jq '.access_token | split(".") | .[1] | @base64d | fromjson'
{
  "aud": [],
  "client_id": "my-client",
  "exp": 1670354213,
  "iat": 1670354113,
  "iss": "https://dmv.infratographer.com/",
  "jti": "e36322d3-414c-4da2-91a8-f19a6e9fb1d3",
  "scp": [],
  "sub": "my-user-id"
}
```

[jq]: https://stedolan.github.io/jq/

### Claim mapping

DMV supports mapping of subject token claims to issued token claims using [CEL][cel]. `oauth.claimMappings` in the config file defines the mappings of issued token claims to CEL expressions. For example, the following config snippet will map the `infratographer:group` claim based on the `sub` claim:

```yaml
oauth:
  claimMappings:
    "infratographer:group": "claims.sub == '1234' ? 'admin' : 'user'"
```

The following variables are predefined in the CEL runtime environment:

* `claims`: A dynamic map containing all claims in the subject token JWT. Accessible using `claims.*` (e.g., `claims.sub`)
* `subSHA256`: The hex-encoded SHA256 sum of the `sub` claim

[cel]: https://github.com/google/cel-go

### JWKS

The [JSON Web Key Set][jwks] (JWKS) used for signing dmv JWTs is available at `/jwks.json`.

[jwks]: https://www.rfc-editor.org/rfc/rfc7517.html#section-5

### Configuration

dmv requires a configuration file to run. An example can be found at `dmv.example.yaml`.

Private keys must be explicitly configured with a JWT signing algorithm, such as HS256 or RS256. Symmetric keys are loaded from key files as raw bytes. All asymmetric (i.e., RSA) signing keys must be encoded using [PKCS #8][pkcs8]. To generate an RSA private key for development, the following command should get you started:

```
$ openssl genpkey -out privkey.pem -algorithm RSA -pkeyopt rsa_keygen_bits:4096
```

Update the config file and/or Docker Compose volume mounts accordingly.

[pkcs8]: https://en.wikipedia.org/wiki/PKCS_8

## Development

DMV includes a [Dev Container][dev-container] for facilitating service development. Using the Dev Container is not required, but provides a consistent environment for all contributors as well as a few perks like:

* [gopls][gopls] integration out of the box
* Host SSH auth socket mount
* Git support

To get started, you can use either [VS Code][vs-code] or the official [CLI][cli].

[dev-container]: https://containers.dev/
[gopls]: https://pkg.go.dev/golang.org/x/tools/gopls
[vs-code]: https://code.visualstudio.com/docs/devcontainers/containers
[cli]: https://github.com/devcontainers/cli

### Manually setting up SSH agent forwarding

In order to perform Git operations (i.e., committing code in the container), you will need to enable SSH agent forwarding from your machine to the Dev Container. While VS Code handles this automatically, for other editors you will need to set this up manually.

To do so, update your `~/.ssh/config` to support agent forwarding and `ControlMaster`. The following config snippet should accomplish this for you (remember to create `~/.ssh/sockets` first):

```
Host YOUR_HOST_HERE
  ForwardAgent yes
  ControlMaster auto
  ControlPersist yes
  ControlPath ~/.ssh/sockets/%C
```

See the man page for `ssh_config` for more information on what these options do.

Following an initial connection (i.e., via a terminal) to start your Dev Container, `SSH_AUTH_SOCK` should be passed through from the container's host to the container itself.
