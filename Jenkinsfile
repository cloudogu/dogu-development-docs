#!groovy

@Library('github.com/cloudogu/ces-build-lib@1.64.0')
import com.cloudogu.ces.cesbuildlib.*

node('docker') {
    timestamps {
        stage('Checkout') {
            checkout scm
        }

        stage('Check Markdown Links') {
            Markdown markdown = new Markdown(this, "3.11.0")
            markdown.check()
        }
    }
}
