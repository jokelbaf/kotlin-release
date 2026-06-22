# kotlin-release

Automatically creates GitHub releases for Kotlin/JVM projects when the `version` in `build.gradle.kts` is updated. Supports publishing to Maven Central as well.

## Usage

```yaml
name: Release
on:
  push:
    branches: [main]
    paths:
      - 'build.gradle.kts'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: jokelbaf/kotlin-release@master
```

The action reads the top-level `version = "x.y.z"` line from `build.gradle.kts` and compares it against the latest git tag. If they differ, a new GitHub release is created with an auto-generated changelog.

## Publishing to Maven Central

### 1. Set up Sonatype

Create an account at [central.sonatype.com](https://central.sonatype.com) and generate a **user token** (Account > Generate User Token). Save the username and password - these become your `MAVEN_USERNAME` and `MAVEN_PASSWORD` secrets.

### 2. Set up a GPG key

Maven Central requires all artifacts to be signed.

```bash
# Generate a key
gpg --gen-key

# Find your key ID
gpg --list-secret-keys --keyid-format SHORT

# Upload your public key so consumers can verify signatures
gpg --keyserver keyserver.ubuntu.com --send-keys YOUR_KEY_ID

# Export your private key
gpg --export-secret-keys --armor YOUR_KEY_ID
```

Add the following secrets to your GitHub repository:

| Secret | Value |
|---|---|
| `GPG_PRIVATE_KEY` | Output of the export command above |
| `GPG_PASSWORD` | The passphrase you chose |
| `MAVEN_USERNAME` | Sonatype token username |
| `MAVEN_PASSWORD` | Sonatype token password |

### 3. Configure Gradle

Add the [vanniktech Maven Publish Plugin](https://github.com/vanniktech/gradle-maven-publish-plugin) to your `build.gradle.kts`:

```kotlin
plugins {
    id("com.vanniktech.maven.publish") version "0.37.0"
}

mavenPublishing {
    publishToMavenCentral()
    signAllPublications()

    coordinates("com.example", "my-library", version.toString())

    pom {
        name = "My Library"
        description = "..."
        url = "https://github.com/example/my-library"
        licenses {
            license {
                name = "MIT License"
                url = "https://opensource.org/licenses/MIT"
            }
        }
        developers {
            developer {
                id = "yourname"
                name = "Your Name"
            }
        }
        scm {
            url = "https://github.com/example/my-library"
        }
    }
}
```

The plugin reads `signingKey`, `signingPassword`, `mavenCentralUsername`, and `mavenCentralPassword` from Gradle project properties, which this action sets automatically via environment variables.

### 4. Update your workflow

```yaml
      - uses: jokelbaf/kotlin-release@master
        with:
          gpg_private_key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg_password: ${{ secrets.GPG_PASSWORD }}
          maven_username: ${{ secrets.MAVEN_USERNAME }}
          maven_password: ${{ secrets.MAVEN_PASSWORD }}
```

## Inputs

| Input | Description | Default |
|---|---|---|
| `gpg_private_key` | ASCII-armored GPG private key | - |
| `gpg_password` | GPG key passphrase | - |
| `maven_username` | Sonatype Central Portal token username | - |
| `maven_password` | Sonatype Central Portal token password | - |
| `publish_task` | Gradle task used to publish artifacts | `publishToMavenCentral` |
| `java_version` | Java version to use when publishing | `17` |
| `gradle_file` | Path to `build.gradle.kts` | `build.gradle.kts` |
| `cliff_template` | URL to a custom git-cliff template | built-in template |

## Acknowledgements

[Seria's](https://github.com/seriaati) [create-release](https://github.com/seriaati/create-release) action (for python) was used as a reference for this action.
