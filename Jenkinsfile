def notify(status){
	emailext (
		body: '$DEFAULT_CONTENT',
		recipientProviders: [
			[$class: 'CulpritsRecipientProvider'],
			[$class: 'DevelopersRecipientProvider'],
			[$class: 'RequesterRecipientProvider']
		],
		replyTo: '$DEFAULT_REPLYTO',
		subject: '$DEFAULT_SUBJECT',
		to: '$DEFAULT_RECIPIENTS'
	)
}

@NonCPS
def killall_jobs() {
	def jobname = env.JOB_NAME;
	def buildnum = env.BUILD_NUMBER.toInteger();
	def killnums = "";
	def job = Jenkins.instance.getItemByFullName(jobname);
	def split_job_name = env.JOB_NAME.split(/\/{1}/);
	def fixed_job_name = split_job_name[1].replace('%2F',' ');

	for (build in job.builds) {
		if (!build.isBuilding()) { continue; }
		if (buildnum == build.getNumber().toInteger()) { continue; println "equals"; }
		if (buildnum < build.getNumber().toInteger()) { continue; println "newer"; }

		echo("Kill task = ${build}");

		killnums += "#" + build.getNumber().toInteger() + ", ";

		build.doStop();
	}

	if (killnums != "") {
		discordSend description: "in favor of #${buildnum}, ignore following failed builds for ${killnums}", footer: "", link: env.BUILD_URL, result: "ABORTED", title: "[${split_job_name[0]}] Killing task(s) ${fixed_job_name} ${killnums}", webhookURL: env.GS2EMU_WEBHOOK
	}
	echo("Done killing");
}

def buildStepDocker() {
	def split_job_name = env.JOB_NAME.split(/\/{1}/);
	def fixed_job_name = split_job_name[1].replace('%2F',' ');

	try {
		checkout(scm);

		def buildenv = "";
		def tag = '';
		def VER = '';
		def EXTRA_VER = '';


		if(env.TAG_NAME) {
			sh(returnStdout: true, script: "echo '```' > RELEASE_DESCRIPTION.txt");
			env.RELEASE_DESCRIPTION = sh(returnStdout: true, script: "git tag -l --format='%(contents)' ${env.TAG_NAME} >> RELEASE_DESCRIPTION.txt");
			sh(returnStdout: true, script: "echo '```' >> RELEASE_DESCRIPTION.txt");
		}

		if (env.BRANCH_NAME.equals('main')) {
			tag = "latest";
		} else {
			tag = "${env.BRANCH_NAME.replace('/','-')}";
		}

		def PUSH_ARTIFACT = false;

		if (env.TAG_NAME) {
			EXTRA_VER = "";
			VER = "${env.TAG_NAME}";
			PUSH_ARTIFACT = true;
		} else if (env.BRANCH_NAME.equals('dev')) {
			EXTRA_VER = "--build-arg VER_EXTRA=-beta";
		} else {
			EXTRA_VER = "--build-arg VER_EXTRA=-${tag}";
		}

		docker.withRegistry("https://index.docker.io/v1/", "dockergraal") {
			def release_name = env.JOB_NAME.replace('%2F','/');
			def release_type = ("${release_name}").replace('/','-').replace('node-grc-','').replace('main','').replace('dev','');

			def customImage;

			stage("Building project") {
				customImage = docker.build("ghidra:${tag}", "--build-arg BUILDENV=${buildenv} ${EXTRA_VER} --network=host --pull -f Dockerfile .");
			}

			if (PUSH_ARTIFACT) {
				stage("Archiving artifacts...") {
					customImage.inside("") {
						sh "mkdir -p ./dist && cp -fvr /home/gradle/src/build/dist/* ./dist"

						dir("./dist") {
							sh "unzip -j ghidra_*.zip */Extensions/Ghidra/*.zip"
							archiveArtifacts artifacts: '*.zip,*.tar.gz,*.tgz', allowEmptyArchive: true
							//discordSend description: "Docker Image: ${DOCKER_ROOT}/${DOCKERIMAGE}:${tag}", footer: "", link: env.BUILD_URL, result: currentBuild.currentResult, title: "[${split_job_name[0]}] Artifact Successful: ${fixed_job_name} #${env.BUILD_NUMBER}", webhookURL: env.GS2EMU_WEBHOOK;
						}
					}
					def dockerImageRef = docker.image("amigadev/docker-base:latest");
					dockerImageRef.pull();

					dockerImageRef.inside("") {

						stage("Github Release") {
							withCredentials([string(credentialsId: 'PREAGONAL_GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
								dir("./dist") {
									if (!env.CHANGE_ID) { // Don't run on PR's
										def release_type_tag = 'develop';
										def pre_release = '--pre-release';
										if (env.TAG_NAME) {
											pre_release = '';
											release_type_tag = env.TAG_NAME;
										} else if (env.BRANCH_NAME.equals('master')) {
											release_type_tag = 'nightly';
										}


										if (!env.TAG_NAME) {
											sh(returnStdout: true, script: "echo -e '${release_type_tag} releases' > ../RELEASE_DESCRIPTION.txt");
										}

										def files = sh(returnStdout: true, script: 'find . -name "*.zip" -o -name "*.tar.gz"').split('\n');

										try {
											sh "cat ../RELEASE_DESCRIPTION.txt | github-release release --user Preagonal --repo ghidra --tag ${release_type_tag} --name \"Ghidra ${release_type_tag}\" ${pre_release} --description -"
										} catch(err) {

										}

										files.eachWithIndex { file, idx -> 
											file = sh (script: "basename ${file}",returnStdout:true).trim();
											try {
												sh "github-release upload --user Preagonal --repo ghidra --tag ${release_type_tag} --name \"${file}\" --file ${file} --replace";
											} catch(err) {
												sleep 15;
												sh "github-release upload --user Preagonal --repo ghidra --tag ${release_type_tag} --name \"${file}\" --file ${file} --replace";
											}
										}​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​
									}
								}
							}
						}
					}
				}
			} else {
				// Do nothing
			}

			def archive_date = sh (
				script: 'date +"-%Y%m%d-%H%M"',
				returnStdout: true
			).trim();

			if (env.TAG_NAME) {
				archive_date = '';
			}

			if (env.TAG_NAME) {

			}
		}
	} catch(err) {
		currentBuild.result = 'FAILURE'
		discordSend description: "", footer: "", link: env.BUILD_URL, result: currentBuild.currentResult, title: "[${split_job_name[0]}] Build Failed: ${fixed_job_name} #${env.BUILD_NUMBER}", webhookURL: env.GS2EMU_WEBHOOK

		notify("Build Failed: ${fixed_job_name} #${env.BUILD_NUMBER}")
		throw err
	}
}

node('master') {
    killall_jobs();
	def split_job_name = env.JOB_NAME.split(/\/{1}/);
	def fixed_job_name = split_job_name[1].replace('%2F',' ');
	checkout(scm);

	env.COMMIT_MSG = sh(
		script: 'git log -1 --pretty=%B ${GIT_COMMIT}',
		returnStdout: true
	).trim();

	env.GIT_COMMIT = sh(
		script: 'git log -1 --pretty=%H ${GIT_COMMIT}',
		returnStdout: true
	).trim();

	//sh('git fetch --tags');

	env.LATEST_TAG = sh(
		script: 'git tag --sort=creatordate -l | tail -1',
		returnStdout: true
	).trim();

	echo("Latest tag: ${env.LATEST_TAG}");

	def version = env.LATEST_TAG.split(/\./);

	echo("Version: ${version}");

	def verMajor = version[0] as Integer;
	def verMinor = version[1] as Integer;
	def verPatch = version[2] as Integer;
	def verRev = version[3] as Integer;
	def versionChanged = false;

	echo("Version - Major: ${verMajor}, Minor: ${verMinor}, Patch: ${verPatch}");

	if (env.BRANCH_NAME.equals('main')) {
		verMinor++;
		verPatch = 0;
		versionChanged = true;
	} else if (env.BRANCH_NAME.equals('dev')) {
		verPatch++;
		versionChanged = true;
	} else if (env.BRANCH_NAME.equals('feature/preagonal-changes')) {
		verRev++;
		versionChanged = true;
	}

	if (versionChanged) {
		withCredentials([string(credentialsId: 'PREAGONAL_GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
			def tagName = "${verMajor}.${verMinor}.${verPatch}.${verRev}";

			def iso8601Date = sh(
				script: 'date -Iseconds',
				returnStdout: true
			).trim();

			env.JSON_RESPONSE = sh(
				script: "curl -L -X POST -H \"Accept: application/vnd.github+json\" -H \"Authorization: Bearer ${env.GITHUB_TOKEN}\" -H \"X-GitHub-Api-Version: 2022-11-28\" https://api.github.com/repos/Preagonal/ghidra/git/tags -d '{\"tag\":\"${tagName}\",\"message\":\"${env.COMMIT_MSG}\",\"object\":\"${env.GIT_COMMIT}\",\"type\":\"tree\",\"tagger\":{\"name\":\"preagonal-pipeline[bot]\",\"email\":\"119898225+preagonal-pipeline[bot]@users.noreply.github.com\",\"date\":\"${iso8601Date}\"}}'",
				returnStdout: true
			);
			def response = readJSON(text: env.JSON_RESPONSE);

			sh(
				script: "curl -L -X POST -H \"Accept: application/vnd.github+json\" -H \"Authorization: Bearer ${env.GITHUB_TOKEN}\" -H \"X-GitHub-Api-Version: 2022-11-28\" https://api.github.com/repos/Preagonal/ghidra/git/refs -d '{\"ref\": \"refs/tags/${tagName}\", \"sha\": \"${response.sha}\"}'",
				returnStdout: true
			);
		}
	}

	discordSend description: "${env.COMMIT_MSG}", footer: "", link: env.BUILD_URL, result: currentBuild.currentResult, title: "[${split_job_name[0]}] Build Started: ${fixed_job_name} #${env.BUILD_NUMBER}", webhookURL: env.GS2EMU_WEBHOOK

	if (env.TAG_NAME) {
		sh(returnStdout: true, script: "echo '```' > RELEASE_DESCRIPTION.txt");
		env.RELEASE_DESCRIPTION = sh(returnStdout: true, script: "git tag -l --format='%(contents)' ${env.TAG_NAME} >> RELEASE_DESCRIPTION.txt");
		sh(returnStdout: true, script: "echo '```' >> RELEASE_DESCRIPTION.txt");
	}

	node("linux") {
		buildStepDocker();
	}

	if (env.TAG_NAME) {
		//def DESC = sh(returnStdout: true, script: 'cat RELEASE_DESCRIPTION.txt');
	}

	sh("rm -rf ./*");
}
