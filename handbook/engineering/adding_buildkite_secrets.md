# Adding or changing Buildkite secrets

This page outlines the process for adding or changing secrets available on our Buildkite agents.

## Adding secrets to the Buildkite agent environment

- Add the secret to the [Buildkite agent deployment](https://github.com/sourcegraph/infrastructure/blob/master/kubernetes/ci/buildkite/buildkite-agent/buildkite-agent.Deployment.yaml)
- Commit the secret to the master branch
- Follow the [instructions](https://github.com/sourcegraph/infrastructure/tree/master/kubernetes/ci) to deploy the updated configuration to the cluster
