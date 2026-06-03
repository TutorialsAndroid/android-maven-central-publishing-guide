# Publish Android Library to Maven Central Using Sonatype Central Portal

A complete practical guide to publish an **Android library** to **Maven Central** using:

* Gradle
* `maven-publish`
* GPG signing
* Sonatype Central Portal
* Maven Local publishing
* Central Portal bundle ZIP upload
* PowerShell bundle generation script

Once your library is available on Maven Central, users can install it directly using:

```gradle
implementation 'io.github.yourusername:yourlibrary:1.0.0'
```

---

## Final Publishing Flow

```text
Android Library Project
        ↓
Gradle Maven Publish
        ↓
Signed Maven Local Artifacts
        ↓
Central Portal Bundle ZIP
        ↓
Upload to Maven Central
        ↓
Library Available for Users
```

---

## Step 1: Prepare Your Library Details

In your Android library module, define your Maven publishing information.

Example:

```gradle
ext {
    PUBLISH_GROUP_ID = 'io.github.yourusername'
    PUBLISH_VERSION = '1.0.0'
    PUBLISH_ARTIFACT_ID = 'yourlibrary'

    PUBLISH_DESCRIPTION = 'yourLibrary description'
    PUBLISH_URL = 'https://github.com/yourUsername/yourLibraryName'

    PUBLISH_LICENSE_NAME = 'Apache License'
    PUBLISH_LICENSE_URL = 'https://github.com/yourUsername/yourLibraryName/blob/master/LICENSE'

    PUBLISH_DEVELOPER_ID = 'yourUsername'
    PUBLISH_DEVELOPER_NAME = 'Your Name'
    PUBLISH_DEVELOPER_EMAIL = 'youremail@example.com'

    PUBLISH_SCM_CONNECTION = 'scm:git:github.com/yourUsername/yourLibraryName.git'
    PUBLISH_SCM_DEVELOPER_CONNECTION = 'scm:git:ssh://github.com/yourUsername/yourLibraryName.git'
    PUBLISH_SCM_URL = 'https://github.com/yourUsername/yourLibraryName/tree/master'
}
```

Update these values according to your own library:

```text
PUBLISH_GROUP_ID
PUBLISH_VERSION
PUBLISH_ARTIFACT_ID
PUBLISH_DESCRIPTION
PUBLISH_URL
PUBLISH_DEVELOPER_NAME
PUBLISH_DEVELOPER_EMAIL
PUBLISH_SCM_URL
```

Example dependency:

```gradle
implementation 'io.github.yourUsername:yourLibraryName:1.0.0'
```

---

## Step 2: Configure Root `build.gradle`

Android projects can use two Gradle styles.

Older projects usually use:

```gradle
buildscript {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }

    dependencies {
        classpath 'com.android.tools.build:gradle:9.1.0'
    }
}

apply from: "${rootDir}/scripts/publish-root.gradle"
```

New Android Studio projects usually use the modern `plugins` block with Version Catalog:

```gradle
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.android.library) apply false
}
```

If your project already uses this new format, keep it as it is.

Then add this line below the `plugins {}` block:

```gradle
apply from: "${rootProject.projectDir}/scripts/publish-root.gradle"
```

Final root `build.gradle` example:

```gradle
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.android.library) apply false
}

apply from: "${rootProject.projectDir}/scripts/publish-root.gradle"
```

This line loads publishing and signing credentials from:

```text
local.properties
```

The credentials include:

```properties
ossrhUsername=
ossrhPassword=
signing.keyId=
signing.password=
signing.secretKeyRingFile=
```

If your project uses `settings.gradle` or `settings.gradle.kts`, make sure repositories are available there.

Example `settings.gradle`:

```gradle
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}

rootProject.name = 'YourProjectName'
include ':app'
include ':library'
```

If you use Version Catalog, your Android Gradle Plugin versions are usually defined in:

```text
gradle/libs.versions.toml
```

Example:

```toml
[versions]
agp = "9.1.0"

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
```

Recommended project structure:

```text
root build.gradle              -> applies common scripts
settings.gradle                -> manages repositories
libs.versions.toml             -> manages plugin/dependency versions
library/build.gradle           -> configures Android library and Maven publishing
scripts/publish-root.gradle    -> loads credentials
scripts/publish-module.gradle  -> creates Maven publication
```

---

## Step 3: Configure Gradle Wrapper

Open:

```text
gradle/wrapper/gradle-wrapper.properties
```

Use a Gradle version compatible with your Android Gradle Plugin.

Example:

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-9.4.1-all.zip
```

---

## Step 4: Configure `library/build.gradle`

Example `library/build.gradle`:

```gradle
apply plugin: 'com.android.library'

ext {
    PUBLISH_GROUP_ID = 'io.github.yourusername'
    PUBLISH_VERSION = '1.0.0'
    PUBLISH_ARTIFACT_ID = 'yourlibrary'

    PUBLISH_DESCRIPTION = 'yourLibrary description'
    PUBLISH_URL = 'https://github.com/yourUsername/yourLibraryName'

    PUBLISH_LICENSE_NAME = 'Apache License'
    PUBLISH_LICENSE_URL = 'https://github.com/yourUsername/yourLibraryName/blob/master/LICENSE'

    PUBLISH_DEVELOPER_ID = 'yourUsername'
    PUBLISH_DEVELOPER_NAME = 'Your Name'
    PUBLISH_DEVELOPER_EMAIL = 'youremail@example.com'

    PUBLISH_SCM_CONNECTION = 'scm:git:github.com/yourUsername/yourLibraryName.git'
    PUBLISH_SCM_DEVELOPER_CONNECTION = 'scm:git:ssh://github.com/yourUsername/yourLibraryName.git'
    PUBLISH_SCM_URL = 'https://github.com/yourUsername/yourLibraryName/tree/master'
}

android {
    namespace 'com.example.yourlibrary'

    compileSdk 35

    defaultConfig {
        minSdk 19
        targetSdk 34
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    publishing {
        singleVariant("release") {
            withSourcesJar()
            withJavadocJar()
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }

    lint {
        abortOnError false
    }
}

dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')

    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.12.0'

    implementation 'com.github.bumptech.glide:glide:4.16.0'
    annotationProcessor 'com.github.bumptech.glide:compiler:4.16.0'
}

apply from: "${rootProject.projectDir}/scripts/publish-module.gradle"
```

Important publishing block:

```gradle
publishing {
    singleVariant("release") {
        withSourcesJar()
        withJavadocJar()
    }
}
```

This generates:

```text
AAR file
Sources JAR
Javadoc JAR
```

---

## Step 5: Create `scripts/publish-root.gradle`

Create a `scripts` folder in your project root:

```text
scripts/
    publish-root.gradle
    publish-module.gradle
```

Create:

```text
scripts/publish-root.gradle
```

Add:

```gradle
ext["signing.keyId"] = ''
ext["signing.password"] = ''
ext["signing.secretKeyRingFile"] = ''

ext["ossrhUsername"] = ''
ext["ossrhPassword"] = ''

File secretPropsFile = project.rootProject.file('local.properties')

if (secretPropsFile.exists()) {
    Properties p = new Properties()

    new FileInputStream(secretPropsFile).withCloseable { is ->
        p.load(is)
    }

    p.each { name, value ->
        ext[name] = value
    }
} else {
    ext["ossrhUsername"] = System.getenv('OSSRH_USERNAME') ?: ''
    ext["ossrhPassword"] = System.getenv('OSSRH_PASSWORD') ?: ''

    ext["signing.keyId"] = System.getenv('SIGNING_KEY_ID') ?: ''
    ext["signing.password"] = System.getenv('SIGNING_PASSWORD') ?: ''
    ext["signing.secretKeyRingFile"] = System.getenv('SIGNING_SECRET_KEY_RING_FILE') ?: ''
}
```

This script loads credentials from:

```text
local.properties
```

or from environment variables.

---

## Step 6: Create `scripts/publish-module.gradle`

Create:

```text
scripts/publish-module.gradle
```

Add:

```gradle
apply plugin: 'maven-publish'
apply plugin: 'signing'

group = PUBLISH_GROUP_ID
version = PUBLISH_VERSION

afterEvaluate {
    publishing {
        publications {
            release(MavenPublication) {
                from components.release

                groupId = PUBLISH_GROUP_ID
                artifactId = PUBLISH_ARTIFACT_ID
                version = PUBLISH_VERSION

                pom {
                    packaging = 'aar'

                    name = PUBLISH_ARTIFACT_ID
                    description = PUBLISH_DESCRIPTION
                    url = PUBLISH_URL

                    licenses {
                        license {
                            name = PUBLISH_LICENSE_NAME
                            url = PUBLISH_LICENSE_URL
                        }
                    }

                    developers {
                        developer {
                            id = PUBLISH_DEVELOPER_ID
                            name = PUBLISH_DEVELOPER_NAME
                            email = PUBLISH_DEVELOPER_EMAIL
                        }
                    }

                    scm {
                        connection = PUBLISH_SCM_CONNECTION
                        developerConnection = PUBLISH_SCM_DEVELOPER_CONNECTION
                        url = PUBLISH_SCM_URL
                    }
                }
            }
        }
    }

    signing {
        required {
            gradle.taskGraph.allTasks.any { task ->
                task.name.toLowerCase().contains("publish")
            }
        }

        sign publishing.publications.release
    }
}

ext["signing.keyId"] = rootProject.ext["signing.keyId"]
ext["signing.password"] = rootProject.ext["signing.password"]
ext["signing.secretKeyRingFile"] = rootProject.ext["signing.secretKeyRingFile"]
```

Important:

```gradle
packaging = 'aar'
```

This tells Maven Central that the main artifact is an Android AAR library.

---

## Step 7: Create Sonatype Central Portal Token

Open:

```text
https://central.sonatype.com
```

Login to your Sonatype account.

Open:

```text
Account > User Token
```

Generate a new user token.

You will receive:

```text
Username
Password
```

These are publishing token credentials.

Add them to your root `local.properties`:

```properties
ossrhUsername=YOUR_TOKEN_USERNAME
ossrhPassword=YOUR_TOKEN_PASSWORD
```

Example `local.properties`:

```properties
sdk.dir=C\:\\Users\\HP-User1\\AppData\\Local\\Android\\Sdk

ossrhUsername=YOUR_TOKEN_USERNAME
ossrhPassword=YOUR_TOKEN_PASSWORD

signing.keyId=YOUR_SIGNING_KEY_ID
signing.password=YOUR_SIGNING_PASSWORD
signing.secretKeyRingFile=C\:\\Users\\HP-User1\\.gnupg\\secring.gpg
```

---

## Step 8: Create GPG Signing Key

Maven Central requires signed artifacts.

Install GPG on Windows using Gpg4win.
https://www.gpg4win.org/thanks-for-download.html

Check GPG:

```powershell
gpg --version
```

Create a new GPG key:

```powershell
gpg --full-generate-key
```

Recommended options:

```text
Key type: RSA and RSA
Key size: 4096
Expiry: 0
```

Now GPG will ask for your identity:

```text
Real name: Your Name
Email address: youremail@example.com
Comment:
```

You can leave comment empty.

Then GPG will show:

```text
You selected this USER-ID:
"Your Name <youremail@example.com>"
```

Confirm by entering:

```text
O
```

Now GPG will open a passphrase window.

Enter a strong passphrase and repeat it.

Example:

```text
Passphrase: YourStrongGpgPassword
Repeat: YourStrongGpgPassword
```

This passphrase protects your private signing key.

The same passphrase is used later in `local.properties`:

```properties
signing.password=YourStrongGpgPassword
```

After creating the key, list your secret keys:

```powershell
gpg --list-secret-keys --keyid-format=short
```

You will see something like:

```text
sec   rsa4096/9D49CDF0
uid           [ultimate] Your Name <youremail@example.com>
```

Here:

```text
9D49CDF0
```

is your GPG key ID.

Use it in `local.properties`:

```properties
signing.keyId=9D49CDF0
```

Export the secret key ring file:

```powershell
gpg --export-secret-keys -o C:\Users\HP-User1\.gnupg\secring.gpg
```

Upload your public key to a keyserver:

```powershell
gpg --keyserver keyserver.ubuntu.com --send-keys 9D49CDF0
```

Replace `9D49CDF0` with your own key ID.

Final GPG config in `local.properties`:

```properties
signing.keyId=9D49CDF0
signing.password=YourStrongGpgPassword
signing.secretKeyRingFile=C:\\Users\\HP-User1\\.gnupg\\secring.gpg
```

Important:

```text
Do not share your GPG password.
Do not upload secring.gpg to GitHub.
Do not commit local.properties.
```

Add to `.gitignore`:

```gitignore
local.properties
*.gpg
```

---

## Step 9: Add Signing Details to `local.properties`

Open:

```text
local.properties
```

Add:

```properties
ossrhUsername=YOUR_TOKEN_USERNAME
ossrhPassword=YOUR_TOKEN_PASSWORD

signing.keyId=YOUR_GPG_KEY_ID
signing.password=YOUR_GPG_PASSWORD
signing.secretKeyRingFile=C:\\Users\\HP-User1\\.gnupg\\secring.gpg
```

Example:

```properties
ossrhUsername=abc123token
ossrhPassword=xyz456tokenpassword

signing.keyId=9D49CDF0
signing.password=YourGpgPassword
signing.secretKeyRingFile=C:\\Users\\HP-User1\\.gnupg\\secring.gpg
```

Important: never commit `local.properties`.

Add this to `.gitignore`:

```gitignore
local.properties
*.gpg
```

---

## Step 10: Publish to Maven Local First

Before uploading to Maven Central, publish locally first.

Run:

```powershell
./gradlew clean :library:publishReleasePublicationToMavenLocal
```

After success, check:

```text
C:\Users\HP-User1\.m2\repository\io\github\yourUsername\yourLibraryName\1.0.0
```

You should see:

```text
yourLibraryName-1.0.0.aar
yourLibraryName-1.0.0.aar.asc
yourLibraryName-1.0.0.pom
yourLibraryName-1.0.0.pom.asc
yourLibraryName-1.0.0-sources.jar
yourLibraryName-1.0.0-sources.jar.asc
yourLibraryName-1.0.0-javadoc.jar
yourLibraryName-1.0.0-javadoc.jar.asc
yourLibraryName-1.0.0.module
yourLibraryName-1.0.0.module.asc
```

If `.asc` files are generated, signing is working.

---

## Step 11: Create Central Portal Bundle Script

Create:

```text
create-central-bundle.ps1
```

Add:

```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$Version
)

$groupPath = "io\github\yourGithubUserName\yourLibraryName"
$repo = "$env:USERPROFILE\.m2\repository"
$source = "$repo\$groupPath\$Version"

$bundle = "$env:USERPROFILE\Desktop\yourLibraryName-central-bundle"
$dest = "$bundle\$groupPath\$Version"
$zip = "$env:USERPROFILE\Desktop\yourLibraryName-$Version-central.zip"

if (!(Test-Path $source)) {
    Write-Host "ERROR: Maven local version folder not found:" -ForegroundColor Red
    Write-Host $source
    exit 1
}

Write-Host "Cleaning old bundle..."
Remove-Item $bundle -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item $zip -Force -ErrorAction SilentlyContinue

Write-Host "Generating md5 and sha1 checksums..."
Get-ChildItem $source -File | Where-Object {
    $_.Name -notlike "*.md5" -and $_.Name -notlike "*.sha1"
} | ForEach-Object {
    $file = $_.FullName

    $md5 = (Get-FileHash $file -Algorithm MD5).Hash.ToLower()
    $sha1 = (Get-FileHash $file -Algorithm SHA1).Hash.ToLower()

    Set-Content -Path "$file.md5" -Value $md5 -NoNewline
    Set-Content -Path "$file.sha1" -Value $sha1 -NoNewline
}

Write-Host "Creating bundle folder..."
New-Item -ItemType Directory -Force -Path $dest | Out-Null

Write-Host "Copying files..."
Copy-Item "$source\*" $dest -Recurse

Write-Host "Creating ZIP..."
Compress-Archive -Path "$bundle\io" -DestinationPath $zip -Force

Write-Host ""
Write-Host "DONE!" -ForegroundColor Green
Write-Host "Upload this file to Maven Central:"
Write-Host $zip -ForegroundColor Cyan
```

This script:

```text
1. Finds your Maven Local artifact
2. Generates md5 and sha1 checksums
3. Creates correct Maven folder structure
4. Creates a Central Portal upload ZIP
```

---

## Step 12: Generate the Upload ZIP

Run:

```powershell
powershell -ExecutionPolicy Bypass -File .\create-central-bundle.ps1 -Version 1.0.0
```

It will create:

```text
C:\Users\HP-User1\Desktop\yourLibraryName-1.0.0-central.zip
```

Expected ZIP structure:

```text
io/
  github/
    yourGithubUserName/
      yourLibraryName/
        1.0.0/
          yourLibraryName-1.0.0.aar
          yourLibraryName-1.0.0.aar.asc
          yourLibraryName-1.0.0.aar.md5
          yourLibraryName-1.0.0.aar.sha1

          yourLibraryName-1.0.0.pom
          yourLibraryName-1.0.0.pom.asc
          yourLibraryName-1.0.0.pom.md5
          yourLibraryName-1.0.0.pom.sha1

          yourLibraryName-1.0.0-sources.jar
          yourLibraryName-1.0.0-sources.jar.asc
          yourLibraryName-1.0.0-sources.jar.md5
          yourLibraryName-1.0.0-sources.jar.sha1

          yourLibraryName-1.0.0-javadoc.jar
          yourLibraryName-1.0.0-javadoc.jar.asc
          yourLibraryName-1.0.0-javadoc.jar.md5
          yourLibraryName-1.0.0-javadoc.jar.sha1

          yourLibraryName-1.0.0.module
          yourLibraryName-1.0.0.module.asc
          yourLibraryName-1.0.0.module.md5
          yourLibraryName-1.0.0.module.sha1
```

---

## Step 13: Upload to Maven Central

Open:

```text
https://central.sonatype.com
```

Go to:

```text
Publishing Settings > Deployments
```

Click:

```text
Publish Component
```

Upload:

```text
yourLibraryName-1.0.0-central.zip
```

Central Portal will validate the bundle.

After validation, the status will move to:

```text
PUBLISHING
```

Wait for some time and refresh.

Once published, your library will be available from Maven Central.

---

## Step 14: Add Dependency in User Projects

After the release is available, users can add Maven Central:

```gradle
repositories {
    google()
    mavenCentral()
}
```

Then add your library dependency:

```gradle
dependencies {
    implementation 'io.github.yourGithubUserName:yourLibraryName:1.0.0'
}
```

---

## Step 15: Release Flow for Future Versions

For the next release, update your version:

```gradle
PUBLISH_VERSION = '1.0.1'
```

Then run:

```powershell
./gradlew clean :library:publishReleasePublicationToMavenLocal
```

Create the Central Portal ZIP:

```powershell
powershell -ExecutionPolicy Bypass -File .\create-central-bundle.ps1 -Version 1.0.1
```

Upload:

```text
yourLibraryName-1.0.1-central.zip
```

Future releases:

```text
1. Update version
2. Publish to Maven Local
3. Create Central ZIP
4. Upload ZIP to Central Portal
5. Publish
```

---

## Important Release Notes

Maven Central versions are immutable.

Once you publish:

```text
1.0.0
```

you cannot replace the same version again.

For every update, use a new version:

```text
1.0.1
1.1.1
1.2.1
2.0.0
```

Never publish secrets.

Do not commit:

```text
local.properties
secring.gpg
GPG password
Sonatype token
```

Always keep your publishing credentials private.

---

## Conclusion

Publishing an Android library to Maven Central becomes easy once the setup is done correctly.

The complete setup requires:

* Maven publishing configuration
* Proper POM metadata
* AAR packaging
* Sources JAR
* Javadoc JAR
* GPG signing
* Checksum files
* Correct Maven folder structure
* Central Portal upload bundle

After the first successful upload, the process becomes repeatable for every future release.

Final dependency example:

```gradle
implementation 'io.github.yourGithubUsername:yourLibraryName:1.0.0'
```

That is the clean and proper way to publish an Android library to Maven Central.
