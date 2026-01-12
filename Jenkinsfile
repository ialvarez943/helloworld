pipeline {
  agent any

  options {
    skipDefaultCheckout()
  }

  stages {
    stage('Get Code') {
      steps {
        git branch: 'feature_fix_coverage', url: 'https://github.com/ialvarez943/helloworld.git'
        stash name: 'code', includes: '**'
      }
    }

    stage('Static') {
      steps {
        bat 'flake8 --exit-zero --format=pylint app > flake8.out'
        recordIssues(
          sourceCodeRetention: 'LAST_BUILD', 
          tools: [flake8(name: 'Flake8', pattern: 'flake8.out')], 
          qualityGates: [
            [threshold: 8, type: 'TOTAL', unstable: true], 
            [threshold: 10, type: 'TOTAL', unstable: false]
          ],
          enabledForFailure: true
        )
      }
    }

    stage('Security') {
      steps {
        bat 'bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'
        recordIssues(
          tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')], 
          qualityGates:[
            [threshold: 2, type: 'TOTAL', unstable: true], 
            [threshold: 4, type:'TOTAL', unstable: false]
          ],
          enabledForFailure: true
        )
      }
    }

    stage('Test') {
      steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
          unstash name: 'code'
          bat '''
              SET PYTHONPATH=%WORKSPACE%
              coverage run --branch --source=app --omit=app\\__init__.py,app\\api.py -m pytest test\\unit --junitxml=result-test.xml && coverage xml
            '''
        }
      }
      post {
        always {
          stash name: 'coverage-res', includes: 'coverage.xml', allowEmpty: true
          stash name: 'unit-res', includes: 'result-test.xml', allowEmpty: true
        }
      }
    }

    stage('Start Flask') {
      steps {
        bat '''
          SET PYTHONPATH=%WORKSPACE%
          set FLASK_APP=app\\api.py
          start flask run 
      '''
      }
    }

    stage('Rest') {
      steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
          unstash name: 'code'
          bat 'pytest --junitxml=result-rest.xml test\\rest'
          stash name: 'rest-res', includes: 'result-rest.xml'
        }
      }
    }

    stage('Unit') {
      steps {
        unstash name: 'unit-res'
        junit 'result-test.xml'
      }
    }

    stage ('Coverage') {
      steps {
        unstash name: 'coverage-res'
        recordCoverage(
          tools: [
            [parser: 'COBERTURA', pattern: 'coverage.xml']
          ],
          qualityGates: [
            [criticality: 'ERROR', integerThreshold: 85, metric: 'LINE', threshold: 85.0],
            [criticality: 'NOTE', integerThreshold: 95, metric: 'LINE', threshold: 95.0], 
            [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0],
            [criticality: 'NOTE', integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]
          ]
        )
      }
    }

    stage('Performance') {
      steps {
        bat 'D:\\Proyectos\\ESTUDIOS\\UNIR\\DEVOPS\\PRACTICAS\\apache-jmeter-5.6.3\\bin\\jmeter -n -t test\\jmeter\\flask.jmx -f -l flask.jtl'
        perfReport sourceDataFiles: 'flask.jtl'
      }
    }
  }
}