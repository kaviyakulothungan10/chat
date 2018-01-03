# chat
//*************************************************************************
//Template Script for Java->ANT Pipeline Job test Template
//Please check with RBARATHY@travelers.com before updating
//*************************************************************************
node ('TPC1CIU3') {

	stage 'Checkout'
				checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[credentialsId: 'SVN-Credentials', depthOption: 'infinity', ignoreExternalsOption: true, local: './', remote: 'http://scm.prodlb.travp.net/cl/wcplacement/trunk/WCPlacement']], workspaceUpdater: [$class: 'UpdateUpdater']]) 
				def definition = readYaml(file: './deliverydefinition.yml')
		try{
			if("${definition.jenkins.scm.svnurl2}"!='null') {
				checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[credentialsId: 'SVN-Credentials', depthOption: 'infinity', ignoreExternalsOption: true, local: "${definition.jenkins.scm.localcheckoutdirectory2}", remote: "${definition.jenkins.scm.svnurl2}"]], workspaceUpdater: [$class: 'UpdateUpdater']]) 				
			}
			if("${definition.jenkins.scm.svnurl3}"!='null') {
				checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[credentialsId: 'SVN-Credentials', depthOption: 'infinity', ignoreExternalsOption: true, local: "${definition.jenkins.scm.localcheckoutdirectory3}", remote: "${definition.jenkins.scm.svnurl3}"]], workspaceUpdater: [$class: 'UpdateUpdater']])
			}
			def Release = "${definition.general.release_month}"
			def UCDEnv = "${definition.jenkins.deploy.ucd_environment_name}" 
			def service = "${definition.general.job_name}"
			def emailto = "${definition.general.emailto}"
			env.service = "${service}"
			env.release = "${Release}"
			env.UCDEnv =  "${UCDEnv}"
			env.emailto = "${emailto}"
			def buildstatus = "SUCCESS"
			env.buildstatus = "SUCCESS"
		} catch (e) {
				currentBuild.result = "FAILED"
				notifyFailed('Checkout')
				throw e
			
		}
		
	//******************** Defining component version ********************
	
	if ("${definition.jenkins.build.enable}" == "true")
	{	
	stage 'Build'
		try{
			echo "Build Stage Begins"
			
				env.JAVA_HOME=tool name: "${definition.jenkins.build.jdkversion}", type: 'jdk'
				env.ANT_HOME=tool name: "${definition.jenkins.build.antversion}", type: 'ant'
				echo "${env.ANT_HOME}"
                echo "${env.JAVA_HOME}"
			
		if(isUnix()) {
					def temp = "${definition.jenkins.build.buildproperties}".substring(1, "${definition.jenkins.build.buildproperties}".length() -1)
					def build_properties = temp.replaceAll(',','')
					withEnv(["PATH=${env.JAVA_HOME}/bin;${env.ANT_HOME}/bin;${PATH}"]) {
					//echo "{build_properties}"
					sh "${env.ANT_HOME}/bin/ant  -file ${definition.jenkins.build.buildfilepath} -Dbuild.target=${definition.jenkins.build.targets} -DJAVA.HOME=\"${env.JAVA}\" -Dworkspace=${WORKSPACE} ${build_properties} ${definition.jenkins.build.targets}"
						}
						}
				 else {
					def temp = "${definition.jenkins.build.buildproperties}".substring(1, "${definition.jenkins.build.buildproperties}".length() -1)
					def build_properties = temp.replaceAll(',','')
					withEnv(["PATH=${JAVA_HOME}/bin;${ANT_HOME}/bin;${PATH}"]) {
					bat "ant  -file=${definition.jenkins.build.buildfilepath} ${definition.jenkins.build.targets} -DJAVA.HOME=\"${JAVA_HOME}\" -Dworkspace=${WORKSPACE} ${build_properties}"
					}
				}			
			}	

			catch (e) {
				currentBuild.result = "FAILED"
				notifyFailed('Build')
				throw e
			}	
	}
			
	if ("${definition.jenkins.unittests.enable}" == "true")
	{
	stage 'UnitTest'
		try{
			echo "UnitTest Stage Begins"
			if(isUnix()) {
					def temp = "${definition.jenkins.unittests.buildproperties}".substring(1, "${definition.jenkins.unittests.buildproperties}".length() -1)
					def build_properties = temp.replaceAll(',','')
					withEnv(["PATH=${JAVA_HOME}/bin:${ANT_HOME}/bin:${PATH}"]) {
					echo "{build_properties}"
					sh "ant  -file=${definition.jenkins.unittests.buildfilepath} -Dbuild.target=${definition.jenkins.unittests.targets} -DJAVA.HOME=\"${JAVA_HOME}\" -Dworkspace=${WORKSPACE} ${build_properties}"
					}
			}
				 else {
					def temp = "${definition.jenkins.unittests.buildproperties}".substring(1, "${definition.jenkins.unittests.buildproperties}".length() -1)
					def build_properties = temp.replaceAll(',','')
					withEnv(["PATH=${JAVA_HOME}/bin;${ANT_HOME}/bin;${PATH}"]) {
					bat "ant  -file=${definition.jenkins.unittests.buildfilepath} ${definition.jenkins.unittests.targets} -DJAVA.HOME=\"${JAVA_HOME}\" -Dworkspace=${WORKSPACE} ${build_properties}"
					}
				}
				
				if(currentBuild.result == "FAILURE")
			{
					if(isUnix()) 
					 { 
						sh 'exit /b 1'
					 }
					else
					{ 
						bat 'exit /b 1'
					}
			}

		  }catch (e) {
				currentBuild.result = "FAILED"
				notifyFailed('UnitTest')
				throw e
		}
	}

	// ************************ Quality Gates Stage Begins Here ********************
	if ("${definition.jenkins.gates.enable}" == "true")
	{
	stage 'Quality Gates'
		try{
			echo "Quality Gates Stage Begins"
			if("${definition.jenkins.gates.junit}" == "enable") {
			echo "Build Gate: Publishing Test Results!"
			junit healthScaleFactor: 10.0, testResults: './buildAutomation/reports/JunitReports/*.xml'	
			}
			
			if("${definition.jenkins.gates.pmd}" == "enable") {
			echo "Pre-Build Gate: Publishing PMD Results!"
			step([$class: 'PmdPublisher', canComputeNew: false, canRunOnFailed: true, defaultEncoding: '', failedTotalHigh: '10', failedTotalLow: '100', failedTotalNormal: '100', healthy: '', pattern: '**/*_pmd_report.xml', shouldDetectModules: true, unHealthy: ''])
			}
			
			if("${definition.jenkins.gates.cpd}" == "enable") {
			echo "Pre-Build Gate: Publishing CPD Results!"
			dry canComputeNew: false, defaultEncoding: '', failedTotalHigh: '10', failedTotalLow: '100', failedTotalNormal: '100', healthy: '', pattern: '**/*_cpd.xml', unHealthy: ''
			}
			
			if("${definition.jenkins.gates.jacoco}" == "enable") {
			echo "Build Gate: Publishing JaCoCo Coverage Results!"
			jacoco buildOverBuild: true, changeBuildStatus: true, maximumBranchCoverage: '55', maximumClassCoverage: '55', maximumComplexityCoverage: '55', maximumInstructionCoverage: '55', maximumLineCoverage: '55', maximumMethodCoverage: '55', skipCopyOfSrcFiles: true
			}
			
			if("${definition.jenkins.gates.clover}" == "enable") {			
			step([$class: 'CloverPublisher', cloverReportFileName: 'clover.xml', failingTarget: [], healthyTarget: [conditionalCoverage: 80, methodCoverage: 70, statementCoverage: 80], unhealthyTarget: []])
			}
			
			if(currentBuild.result == "FAILURE")
			{
					if(isUnix()) 
					 { 
						sh 'exit /b 1'
					 }
					else
					{ 
						bat 'exit /b 1'
					}
			}
			
		  }catch (e) {
				currentBuild.result = "FAILED"
				notifyFailed('Quality Gates')
				throw e
		}	
	
	}
	else if ("${definition.jenkins.gates.enable}" == "false")
		{
		stage 'Quality Gates'
		echo "No Quality Gates are enabled for this project"
		}

	if ("${definition.jenkins.deploy.enable}" == "true")
	{
		stage 'Deploy'
			echo "Deploy Stage Begins"
			
			if ("${definition.jenkins.deploy.deploy}" == "true")
			{
			echo "enabling Auto Deployment"
				build job: 'LSF/test_MIRA_Pipeline', parameters: [string(name: 'workspace', value: ''), string(name: 'app_svn', value: "${definition.jenkins.scm.svnurl1}"), string(name: 'MIRA_Name', value: "${definition.jenkins.deploy.assetID}"), string(name: 'Version', value: "${definition.jenkins.deploy.version}"), string(name: 'Environment', value: "${definition.jenkins.deploy.environment}"), booleanParam(name: 'SKIP', value: false)]	
				currentBuild.result = "SUCCESS"
				notifySuccess('Deploy')
			}
			
			else
			{
			echo "Skipping auto deployment. Only artifacts will be pushed"
				build job: 'LSF/test_MIRA_Pipeline', parameters: [string(name: 'workspace', value: ''), string(name: 'app_svn', value: "${definition.jenkins.scm.svnurl1}"), string(name: 'MIRA_Name', value: "${definition.jenkins.deploy.assetID}"), string(name: 'Version', value: "${definition.jenkins.deploy.version}"), string(name: 'Environment', value: "${definition.jenkins.deploy.environment}"), booleanParam(name: 'SKIP', value: true)]	
				currentBuild.result = "SUCCESS"
				notifySuccess('Deploy')
			}
	}
	else
	{
		echo "Skipping Auto deployment"

	}

}

//define functions


def notifyFailed(String buildStage) {
  emailext (
        body: '${JELLY_SCRIPT,template="swi.jelly"}', replyTo: '$DEFAULT_REPLYTO', subject:"[${buildStage}] :: A Jenkins Pipeline build failure has been detected for ${env.service}" , to: "${env.emailto}"
    )
    }
	
def notifySuccess(String buildStage) {
  emailext (
        body: '${JELLY_SCRIPT,template="swi.jelly"}', replyTo: '$DEFAULT_REPLYTO', subject: "[${buildStage}]:: ${env.UCDEnv} Env: Package for -${env.service} - Marked Success!!" , to: "${env.emailto}"
    )
    }
		 

