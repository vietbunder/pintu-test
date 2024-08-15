# PTU Test Service

This repository contains the PTU Test Service, a simple Node.js application that is containerized using Docker and deployed to a Kubernetes cluster through a CI/CD pipeline managed by Jenkins.

## Repository Structure

- **Dockerfile**: Defines the Docker image build process for the Node.js service.
- **cicd**: Contains Jenkins pipeline scripts for CI/CD automation.
- **index.js**: Main entry point of the Node.js application with routes including `/health` for Kubernetes probes.
- **kubernetes**: Holds Kubernetes manifests for deploying the service in different environments.
- **package.json**: Contains project metadata, including dependencies and scripts.

## Jenkins Pipeline

### Key Stages:

1. **Environment Setup**:
   - Fetch and verify environment configurations for the pipeline.

2. **Build Docker Image**:
   - Builds and tags the Docker image, then pushes it to a container registry.

3. **Deploy to Kubernetes**:
   - Deploys the service to development, staging, or production environments using Kubernetes manifests.

4. **Notifications**:
   - Sends Slack notifications about the build and deployment status.


## Additional Notes

- The `/health` endpoint is used for Kubernetes liveness and readiness probes.
- The CI/CD pipeline can be extended or customized within the `cicd/` directory.
- Service can be consumed from the public internet using the DNS domain.




https://pintu.development.mofi.id/