# CI Configuration for Dual-Boot

Run your test suite against both dependency sets in CI. These examples use `BUNDLE_GEMFILE=Gemfile.next` to switch between the current and next dependency sets — this works the same whether you are upgrading Rails, Ruby, or another core gem.

---

## GitHub Actions

```yaml
# .github/workflows/ci.yml

name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        gemfile: [Gemfile, Gemfile.next]

    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432

    env:
      BUNDLE_GEMFILE: ${{ matrix.gemfile }}
      DATABASE_URL: postgres://postgres:postgres@localhost/myapp_test
      RAILS_ENV: test

    steps:
      - uses: actions/checkout@v3

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.1
          bundler-cache: true

      - name: Setup Database
        run: |
          bundle exec rails db:create
          bundle exec rails db:schema:load

      - name: Run Tests
        run: bundle exec rspec
```

---

## CircleCI

```yaml
# .circleci/config.yml

version: 2.1

jobs:
  build-current:
    docker:
      - image: cimg/ruby:3.1.0
        environment:
          RAILS_ENV: test
          DATABASE_URL: postgres://postgres@localhost/myapp_test
      - image: cimg/postgres:14.0
    steps:
      - checkout
      - run: bundle install
      - run: bundle exec rails db:create db:schema:load
      - run: bundle exec rspec

  build-next:
    docker:
      - image: cimg/ruby:3.1.0
        environment:
          RAILS_ENV: test
          DATABASE_URL: postgres://postgres@localhost/myapp_test
          BUNDLE_GEMFILE: Gemfile.next
      - image: cimg/postgres:14.0
    steps:
      - checkout
      - run: bundle install
      - run: bundle exec rails db:create db:schema:load
      - run: bundle exec rspec

workflows:
  version: 2
  test:
    jobs:
      - build-current
      - build-next
```

---

## Jenkins

```groovy
// Jenkinsfile

pipeline {
  agent any

  environment {
    RAILS_ENV = 'test'
    DATABASE_URL = 'postgres://postgres:postgres@localhost/myapp_test'
  }

  stages {
    stage('Setup') {
      steps {
        sh 'bundle install'
        sh 'bundle exec rails db:create db:schema:load'
      }
    }

    stage('Test Current') {
      steps {
        sh 'bundle exec rspec'
      }
    }

    stage('Test Next') {
      environment {
        BUNDLE_GEMFILE = 'Gemfile.next'
      }
      steps {
        sh 'bundle install'
        sh 'bundle exec rails db:create db:schema:load'
        sh 'bundle exec rspec'
      }
    }
  }
}
```

---

## Caveats

### Test Suite Duration

Dual-boot CI doubles your test time. Consider:
- Running both versions only on main branch merges
- Running next version tests nightly instead of on every commit
- Parallelizing test runs

### Memory Usage

Running two versions requires more memory. Monitor your CI resources.
