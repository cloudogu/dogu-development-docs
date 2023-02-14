#!groovy

@Library('github.com/cloudogu/ces-build-lib@1.62.0')
import com.cloudogu.ces.cesbuildlib.*

node('docker') {
    timestamps {
        stage('Checkout') {
            checkout scm
        }

        stage('Check Markdown Links') {
            Markdown markdown = new Markdown(this)
            markdown.check()
        }
    }
}
