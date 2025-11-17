# Pipeline Stage Control with Properties Files

## Overview

This repository demonstrates a simple, maintainable pattern for controlling pipeline stage execution using a properties file. This approach provides version-controlled, branch-specific pipeline configuration that's easy to understand and modify.

### Benefits of Properties File Configuration
- **Version Controlled**: Properties file lives in your repository alongside your code
- **Branch Specific**: Each branch can have different stage configurations
- **Code Reviewable**: Changes to pipeline behavior are visible in pull requests
- **Simple and Clear**: Easy to understand what stages will run
- **Repeatable**: Historical builds show exact configuration used
- **Safe Defaults**: Can configure sensible defaults (e.g., prod deployments off by default)

## Use Cases

### Pre-Production Testing Workflows
Enable developers to test changes safely in pre-production environments:
- Feature branches disable production deployments by default
- Developers can enable/disable expensive stages during rapid iteration
- Full validation before merging to main

### Selective Stage Execution
Control which pipeline stages run based on your needs:
- Skip expensive integration tests during rapid development
- Disable security scans for draft PRs
- Control deployment to specific environments per branch

### Multi-Component Builds
For complex builds with multiple components:
- Enable/disable individual component builds
- Coordinate inter-related build processes
- Optimize build times by building only what changed

## Repository Structure

This demonstration uses **two repositories**:

### 1. Application Repository (this repo: `propertiesFileExample`)
```
propertiesFileExample/
├── pipeline.properties    # Stage control configuration
├── myapp/                # Your application code goes here
└── README.md             # This file
```

**Note**: This repository does NOT contain a Jenkinsfile. The Jenkinsfile is centralized in the shared `jenkinsfile-library` repository.

### 2. Jenkinsfile Library Repository
```
jenkinsfile-library/
├── jenkinsfile-propFile-example    # The Jenkinsfile used by this demo
└── (many other shared Jenkinsfiles)
```

**Repository URLs**:
- Application repo: `https://github.com/kmurrayAtCloudbees/propertiesFileExample.git`
- Jenkinsfile repo: `https://github.com/kmurrayAtCloudbees/jenkinsfile-library.git`

## How It Works

### The Marker File Concept

The Multibranch Pipeline uses `pipeline.properties` as a **marker file**. When CloudBees CI scans branches in your application repository:

1. **Branch Discovery**: CloudBees CI scans all branches in the application repo
2. **Marker Check**: For each branch, it checks if `pipeline.properties` exists
3. **Job Creation**: If the marker file exists, a pipeline job is created for that branch
4. **Jenkinsfile Loading**: The job uses the Jenkinsfile from the separate `jenkinsfile-library` repo

**Important**: The marker file name doesn't have to match the properties file purpose. It can be ANY file that identifies "this app should use this pipeline". Examples:
- `pipeline.properties` (what we use)
- `build.gradle` (for Gradle projects)
- `package.json` (for Node.js projects)
- `pom.xml` (for Maven projects)
- `Dockerfile` (for containerized apps)

### The Pipeline Pattern

The Jenkinsfile follows this pattern:

1. **Before Pipeline Block**: Defines defaults and loads properties
   ```groovy
   def defaults = [ /* default values */ ]
   def props = [:]
   
   node {
       checkout scm
       props = readProperties(defaults: defaults, file: 'pipeline.properties')
   }
   ```

2. **Pipeline Stages**: Use `when` conditions to check property values
   ```groovy
   stage('Build') {
       when {
           expression { props['stage.build.enabled'] == 'true' }
       }
       steps {
           // build steps
       }
   }
   ```

3. **Safe Defaults**: Sensible defaults ensure nothing breaks if properties are missing

## Pipeline Properties Configuration

The `pipeline.properties` file controls which stages execute:

```properties
# Build stage - can be disabled for complex multi-component builds
stage.build.enabled=true

# Testing stages - enable/disable based on testing needs
stage.unit.tests.enabled=true
stage.integration.tests.enabled=true

# Security scanning - may want to skip during rapid iteration
stage.security.scan.enabled=true

# Deployment stages - control where artifacts are deployed
stage.deploy.dev.enabled=true
stage.deploy.preprod.enabled=true

# Production deployment - defaults to false for safety
stage.deploy.prod.enabled=false
```

### Modifying Stage Execution

To enable or disable stages, simply edit `pipeline.properties`:

```properties
# Skip expensive tests during rapid development
stage.integration.tests.enabled=false
stage.security.scan.enabled=false

# Deploy only to dev environment for testing
stage.deploy.dev.enabled=true
stage.deploy.preprod.enabled=false
stage.deploy.prod.enabled=false
```

Commit the change, and the next pipeline run will respect your configuration.

## Creating the Multibranch Pipeline Job

> **Note**: This example demonstrates the Multibranch Pipeline pattern. If you want to use properties files with a regular **Pipeline job** (not Multibranch), simply include the `pipeline.properties` file in your application repository and configure the job to get the Jenkinsfile using your normal method (Pipeline script from SCM). The properties file will be available after `checkout scm` executes.

### Option 1: Using Configuration as Code (CasC) YAML

Create a job configuration file with this structure:

```yaml
kind: multibranch
name: propertiesfileExample
description: 'Demonstrates properties-based stage control'
displayName: propertiesfileExample
orphanedItemStrategy:
  defaultOrphanedItemStrategy:
    pruneDeadBranches: true
    daysToKeep: -1
    numToKeep: -1
    abortBuilds: false
projectFactory:
  customBranchProjectFactory:
    marker: pipeline.properties
    definition:
      cpsScmFlowDefinition:
        scriptPath: jenkinsfile-propFile-example
        scm:
          scmGit:
            userRemoteConfigs:
            - userRemoteConfig:
                url: https://github.com/kmurrayAtCloudbees/jenkinsfile-library.git
            branches:
            - branchSpec:
                name: '*/master'
        lightweight: true
properties:
- envVars: {}
sourcesList:
- branchSource:
    source:
      git:
        traits:
        - gitBranchDiscovery: {}
        credentialsId: ''
        id: 62787fc9-441a-4bf2-8a91-23dac1f420fb
        remote: https://github.com/kmurrayAtCloudbees/propertiesFileExample.git
    strategy:
      allBranchesSame: {}
```

### Option 2: Using CloudBees CI UI

1. **Create New Item**
   - Click "New Item"
   - Enter name: `propertiesfileExample`
   - Select "Multibranch Pipeline"
   - Click OK

2. **Configure Branch Sources**
   - Click "Add source" → "Git"
   - Project Repository: `https://github.com/kmurrayAtCloudbees/propertiesFileExample.git`
   - Credentials: (none needed for public repos)
   - Behaviors: Add "Discover branches"

3. **Configure Build Configuration**
   - Mode: "by custom script"
   - Marker file: `pipeline.properties`
   - Script Path (in the custom script SCM section):
     - SCM: Git
     - Repository URL: `https://github.com/kmurrayAtCloudbees/jenkinsfile-library.git`
     - Branch: `*/master`
     - Script Path: `jenkinsfile-propFile-example`
     - Lightweight checkout: ✓ enabled

4. **Save and Scan**
   - Click "Save"
   - CloudBees CI will scan for branches with `pipeline.properties`
   - Jobs will be created for each discovered branch

## Key Configuration Elements Explained

### projectFactory: customBranchProjectFactory

This is the key to using an external Jenkinsfile:

- **marker**: `pipeline.properties`
  - The file that must exist in a branch for a job to be created
  - Can be ANY file - doesn't have to be the actual configuration file
  - Common alternatives: `pom.xml`, `package.json`, `Dockerfile`, `build.gradle`

- **definition: cpsScmFlowDefinition**
  - Points to the separate Jenkinsfile repository
  - **scriptPath**: Must match the filename in the `jenkinsfile-library` repo
  - **scm**: Defines where to find the Jenkinsfile

### sourcesList: branchSource

This defines the application repository to scan:

- **source: git**: The application repository URL
- **traits: gitBranchDiscovery**: Automatically discover all branches
- **strategy: allBranchesSame**: Use the same build configuration for all branches

## Example Workflows

### Feature Branch Development

**Scenario**: Developer working on a new feature, wants fast feedback without expensive stages.

1. Create feature branch: `git checkout -b feature/new-api`
2. Modify `pipeline.properties`:
   ```properties
   stage.build.enabled=true
   stage.unit.tests.enabled=true
   stage.integration.tests.enabled=false        # Skip for speed
   stage.security.scan.enabled=false            # Skip for speed
   stage.deploy.dev.enabled=true
   stage.deploy.preprod.enabled=false           # Not ready yet
   stage.deploy.prod.enabled=false
   ```
3. Commit and push: Pipeline runs quickly with only essential stages
4. Before merging to main, re-enable all stages for full validation

### Pre-Production Testing

**Scenario**: Testing changes in preprod before production release.

Main branch `pipeline.properties`:
```properties
stage.build.enabled=true
stage.unit.tests.enabled=true
stage.integration.tests.enabled=true
stage.security.scan.enabled=true
stage.deploy.dev.enabled=true
stage.deploy.preprod.enabled=true
stage.deploy.prod.enabled=false              # Keep prod safe
```

When ready for production, update the property:
```properties
stage.deploy.prod.enabled=true
```

## Pipeline Stages

The example pipeline includes these stages:

1. **Build**: Compiles/builds the application
2. **Unit Tests**: Runs fast unit tests
3. **Integration Tests**: Runs slower integration tests
4. **Security Scan**: Performs security analysis
5. **Deploy to Dev**: Deploys to development environment
6. **Deploy to Preprod**: Deploys to pre-production environment
7. **Deploy to Production**: Deploys to production environment

Each stage outputs a simple message (e.g., "I built the application") for demonstration purposes. In a real implementation, these would contain your actual build, test, and deployment logic.

## Advanced Patterns

### Branch-Specific Defaults

You can use different default configurations per branch by maintaining different `pipeline.properties` files:

- `main` branch: All stages enabled, prod deployment off
- `develop` branch: All stages enabled, no prod deployment
- `feature/*` branches: Minimal stages, dev deployment only

### Environment-Specific Properties

For more complex scenarios, you could extend this pattern:

```properties
# Base configuration
stage.build.enabled=true

# Environment-specific deployments
stage.deploy.dev.enabled=true
stage.deploy.qa.enabled=true
stage.deploy.staging.enabled=true
stage.deploy.prod.enabled=false

# Integration settings
stage.integration.tests.database.enabled=true
stage.integration.tests.api.enabled=true
```

### Conditional Logic in Pipeline

The Jenkinsfile can implement additional logic beyond simple true/false:

```groovy
stage('Deploy to Production') {
    when {
        allOf {
            expression { props['stage.deploy.prod.enabled'] == 'true' }
            branch 'main'  // Only allow from main branch
        }
    }
    steps {
        echo "I deployed to Production environment"
    }
}
```

## Additional Resources

- **Jenkinsfile Library**: https://github.com/kmurrayAtCloudbees/jenkinsfile-library
- **CloudBees CI Documentation**: https://docs.cloudbees.com/
- **Pipeline Syntax Reference**: https://www.jenkins.io/doc/book/pipeline/syntax/

## Contributing

This is a demonstration repository. Feel free to fork and adapt this pattern for your own use cases.

## License

This example is provided as-is for demonstration purposes.

---