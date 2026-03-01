pipeline {
  agent any
  
  stages {

    stage('Get Code') {
      steps {
        git branch: 'develop', url: 'https://github.com/Minibomba2/todo-list-aws-arb'
        sh '''
          rm -rf _config        
          git clone --depth 1 --branch staging git@github.com:Minibomba2/todo-list-aws-config.git _config
        '''        
        stash name: 'src', includes: '**/*'

      }
    }

    stage('Static Test') {
      steps {
        unstash 'src'
        sh '''

          mkdir -p reports

          ~/.local/bin/flake8 --exit-zero --format=pylint src > reports/flake8.out
          ~/.local/bin/bandit --exit-zero -r src -f custom -o reports/bandit.out \
            --msg-template "{abspath}:{line}: [{test_id}] {msg}"
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
      }
    }

    stage('Deploy') {
      steps {
        unstash 'src'
        sh '''
          sam build
          sam validate
          sam deploy --config-env staging --config-file _config/samconfig.toml \
            --resolve-s3 \
            --no-confirm-changeset --no-fail-on-empty-changeset
        '''
      }
    }

    stage('Rest Test') {
      steps {
        unstash 'src'
        
        sh '''
        set -e
          API_URL=$(aws cloudformation describe-stacks \
            --stack-name "todo-list-aws-staging" \
            --region "us-east-1" \
            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
            --output text)

          echo "API_URL=$API_URL"
          test -n "$API_URL"
          export BASE_URL="$API_URL"
          
          pytest -q test/integration/todoApiTest.py
        '''
      }
    }

    stage('Promote') {
      steps {
        sh '''
          git config user.email "jenkins@local"
          git config user.name "jenkins"
          git fetch origin main
          git checkout main
          git pull --ff-only origin main
          git remote set-url origin git@github.com:Minibomba2/todo-list-aws-arb.git
          git merge --no-ff origin/develop -m "Promote: CI OK"
          git push origin main
        '''
      }
    }
  }
}

