
def indexHost = (env.BRANCH_NAME == 'master') ? "software" : "test-software";

gccVersions = [
    Debian_9: "6",
]

timestamps {
    stage("Pipeline Setup") {
        try {
            def nodes = [["Debian_9", "python3.6"],
                        ["Debian_9", "python2.7"]]

            node {
                step([$class: 'StashNotifier',
                    includeBuildNumberInKey: false])
            }
            currentBuild.result = 'SUCCESS'

            for (nodeCombination in nodes) {
                // Copying to nodeName is necessary due to Groovy internals.
                String nodeName = nodeCombination[0]
                String pythonInterpreter = nodeCombination[1]
                stage("${nodeName} with ${pythonInterpreter}") {
                    node("${nodeName}_internet") {
                        try {
                            scmInfo = checkout scm
                            create_venv(pythonInterpreter)
                            merge_blacklist(pythonInterpreter, nodeName)
                            lock (resource: "oss-list-build-job-${indexHost}", inversePrecedence: true) {
                                build(pythonInterpreter, nodeName, indexHost)
                            }
                            enable_manylinux_wheels(pythonInterpreter)
                            check_whitelist_installable(nodeName, indexHost, scmInfo)
                        } finally {
                            step([$class: 'JUnitResultArchiver', testResults: '*.results.xml'])
                        }
                    }
                }
            }

            stage("Source distributions for pip and virtualenv") {
                node("Debian_9") {
                    checkout scm
                    create_venv("python2.7")
                    lock (resource: "oss-list-build-job-${indexHost}", inversePrecedence: true) {
                        push_pip_and_virtulenv_sdists(indexHost)
                    }
                }
            }
        } catch(err) {
            currentBuild.result = 'FAILED'
            throw err;
        } finally {
            slackNotify{
                slackNotifyChannel='#oss-requests'
                slackNotifyBranches=[]
                slackNotifyIncludeChanges=true
            }
            node {
                step([$class: 'StashNotifier',
                    includeBuildNumberInKey: false])
            }
        }
    }
}

def create_venv(String pythonInterpreter) {
    sh """\
    virtualenv --python=${pythonInterpreter} --no-download /tmp/venv
    cp _manylinux.py /tmp/venv/lib/${pythonInterpreter}/site-packages/
    . /tmp/venv/bin/activate
    """
}

def enable_manylinux_wheels(String pythonInterpreter) {
    sh """#!/bin/bash
    rm -f /tmp/venv/lib/${pythonInterpreter}/site-packages/_manylinux.py{,c,o}
    """
}

def merge_blacklist(String pythonInterpreter, String nodeName) {
    sh 'cp py_build_blacklist.txt py_build_blacklist_merged.txt'
    if (pythonInterpreter == 'python2.7') {
        sh 'cat py_build_blacklist_py2.txt >> py_build_blacklist_merged.txt'
    }
    if (pythonInterpreter == 'python3.4') {
        sh 'cat py_build_blacklist_py3.txt >> py_build_blacklist_merged.txt'
    }
    if (pythonInterpreter == 'python3.6') {
        sh 'cat py_build_blacklist_py3.txt >> py_build_blacklist_merged.txt'
    }
    if (pythonInterpreter == 'python3.7') {
        sh 'cat py_build_blacklist_py3.txt >> py_build_blacklist_merged.txt'
        sh 'cat py_build_blacklist_py37.txt >> py_build_blacklist_merged.txt'
    }
    if (nodeName == "Debian_9") {
        sh 'cat py_build_blacklist_deb9.txt >> py_build_blacklist_merged.txt'
    }
    sh 'echo "merged blacklist"'
    sh 'cat py_build_blacklist_merged.txt'
}

def runInBuildEnv(String devpiHost, String nodeName, String shellCode) {
    def gccVersion = gccVersions[nodeName]

    sh """\
    . /tmp/venv/bin/activate

    # work around broken upstream wheels
    export PIP_NO_BINARY="ptyprocess"
    export PIP_INDEX_URL="${devpiHost}/root/pypi/+simple/"
    export PIP_NO_DEPS="yes"

    # Disable build isolation (with inverted logic as per https://github.com/pypa/pip/issues/5735)
    # Project like Pandas and Cryptography defines a pyproject.toml for PEP 518 compliance. Unfortunately,
    # Pip 10 only partially supports PEP 518; it requires that all dependencies be available as binary
    # distributions which is generally not always the case (see https://github.com/pypa/pip/issues/5171)
    # FIXME: Re-evaluate if still needed after a upgrade of Virtualenv or Pip.
    export PIP_NO_BUILD_ISOLATION="no"

    export CC=gcc-${gccVersion}
    export CXX=g++-${gccVersion}
    export CFLAGS="-g0 -Wl,-strip-all"
    # OSS-518: Ensure we build python-slugify uses text-unidecode instead of Unidecode
    export SLUGIFY_USES_TEXT_UNIDECODE=yes

    ${shellCode}
    """
}

def buildWithDevpi(String pythonInterpreter, String nodeName, String devpiHost) {
    runInBuildEnv(devpiHost, nodeName, """\
        # the following two pip-install commands are necessary for bootstrapping a new index
        pip install --no-deps --index-url ${devpiHost}/for_dev/${nodeName} -r requirements.txt

        pip --version
        easy_install --version
        pip freeze --all
        """
    )

    // We install all packages in a virtualenv containing three golden copies
    // numpy, scipy, pybind11
    //
    // In order to make bootstrapping work, we first build the golden copies
    // with the devpi-builder and upload them to the OSSWhitelist index, then
    // we install them into the virtualenv

    withCredentials([[$class: 'UsernamePasswordMultiBinding',
                      credentialsId: 'oss-list-production-index-credentials',
                      passwordVariable: 'DEVPI_PASSWORD',
                      usernameVariable: 'DEVPI_USER']]) {
        runInBuildEnv(devpiHost, nodeName, """\
            echo "numpy==1.12.1" >> /tmp/numpy_golden_copy.txt
            devpi-builder --batch --blacklist "py_build_blacklist_merged.txt" "/tmp/numpy_golden_copy.txt" "${devpiHost}/${env.DEVPI_USER}/${nodeName}" --pure-index "${devpiHost}/${env.DEVPI_USER}/generic" --junit-xml "golden_numpy.results.xml" --run-id "${nodeName} ${pythonInterpreter} Golden Numpy"
            pip install --index-url="${devpiHost}/${env.DEVPI_USER}/${nodeName}" numpy==1.12.1

            echo "scipy==0.14.1" >> /tmp/scipy_golden_copy.txt
            devpi-builder --batch --blacklist "py_build_blacklist_merged.txt" "/tmp/scipy_golden_copy.txt" "${devpiHost}/${env.DEVPI_USER}/${nodeName}" --pure-index "${devpiHost}/${env.DEVPI_USER}/generic" --junit-xml "golden_scipy.results.xml" --run-id "${nodeName} ${pythonInterpreter} Golden SciPy"
            pip install --index-url="${devpiHost}/${env.DEVPI_USER}/${nodeName}" scipy==0.14.1

            echo "pybind11==2.1.0" >> /tmp/pybind_golden_copy.txt
            devpi-builder --batch --blacklist "py_build_blacklist_merged.txt" "/tmp/pybind_golden_copy.txt" "${devpiHost}/${env.DEVPI_USER}/${nodeName}" --pure-index "${devpiHost}/${env.DEVPI_USER}/generic" --junit-xml "golden_pybind.results.xml" --run-id "${nodeName} ${pythonInterpreter} Golden Pybind"
            pip install --index-url="${devpiHost}/${env.DEVPI_USER}/${nodeName}" pybind11==2.1.0

            pip freeze --all
            """
        )
    }

    try {
        parallel(
            'Production': {
                withCredentials([[$class: 'UsernamePasswordMultiBinding',
                                credentialsId: 'oss-list-production-index-credentials',
                                passwordVariable: 'DEVPI_PASSWORD',
                                usernameVariable: 'DEVPI_USER']]) {
                    runInBuildEnv(devpiHost, nodeName, """\
                        devpi-builder --batch --blacklist "py_build_blacklist_merged.txt" "Production/py_oss_whitelist.txt" "${devpiHost}/${env.DEVPI_USER}/${nodeName}" --pure-index "${devpiHost}/${env.DEVPI_USER}/generic" --junit-xml "production.results.xml" --run-id "${nodeName} ${pythonInterpreter} Production"
                        """
                    )
                }
            },
            'BuildAndDeploy': {
                withCredentials([[$class: 'UsernamePasswordMultiBinding',
                                    credentialsId: 'oss-list-build-and-deploy-index-credentials',
                                    passwordVariable: 'DEVPI_PASSWORD',
                                    usernameVariable: 'DEVPI_USER']]) {
                    runInBuildEnv(devpiHost, nodeName, """\
                        devpi-builder --batch --blacklist "py_build_blacklist_merged.txt" "BuildAndDeploy/py_oss_whitelist.txt" "${devpiHost}/${env.DEVPI_USER}/${nodeName}" --pure-index "${devpiHost}/${env.DEVPI_USER}/generic" --junit-xml "buildanddeploy.results.xml" --run-id "${nodeName} ${pythonInterpreter} BuildAndDeploy"
                        """
                    )
                }
            },
        )
    } finally {
        runInBuildEnv(devpiHost, nodeName, "pip freeze --all")
    }
}

def build(String pythonInterpreter, String nodeName, String indexHost) {
    buildWithDevpi(pythonInterpreter, nodeName, "https://${indexHost}.blue-yonder.org")
}

def push_pip_and_virtulenv_sdists(String indexHost) {
    // sadly this cannot utilize Devpi's push functionality as that would push all release files, not only the source distributions
    def pip_or_setuptools_requirement = /\(^pip==\S\+\)\|\(^setuptools==\S\+\)/
    sh """\
    set -xe
    . /tmp/venv/bin/activate

    mkdir sdists
    grep --only-matching '${pip_or_setuptools_requirement}' Production/py_oss_whitelist.txt | xargs --max-args=1 --no-run-if-empty pip download --no-binary :all: --no-deps --index-url "https://${indexHost}.blue-yonder.org/root/pypi/+simple/" --dest=sdists
    pip install by-devpi-client
    """

    withCredentials([[$class: 'UsernamePasswordMultiBinding',
                      credentialsId: 'oss-list-production-index-credentials',
                      passwordVariable: 'BY_DEVPI_PASSWORD',
                      usernameVariable: 'BY_DEVPI_USER']]) {
        sh """\
        . /tmp/venv/bin/activate

        BY_DEVPI_SERVER=https://${indexHost}.blue-yonder.org by-devpi upload sdists/*
        """
    }
}

/**
 * Verify that all newly added packages are on the whitelist index (with all dependencies).
 *
 * Newly added can mean two different things:
 *  * If this is a branch, it is all packages added since the common ancestor with master
 *  * If this is master, it is all packages not contained in the last successful build
 */
def check_whitelist_installable(String nodeName, String indexHost, def scmInfo) {
    sh """\
#!/tmp/venv/bin/python
# coding=utf-8

from __future__ import print_function, unicode_literals

import os
import re
import subprocess
import sys
import tempfile
import time
import xml.etree.ElementTree

from devpi_builder import requirements


def get_current_branch():
    return os.getenv('BRANCH_NAME')


def write_additions_to_file(diff, filename):
    with open(filename, 'wb') as requirements_file:
        for line in diff.splitlines():
            if line.startswith('+') and not line.startswith('+++'):  # only new lines, not new file marker
                requirements_file.write(line[1:].encode('utf-8'))
                requirements_file.write('\\n'.encode('utf-8'))


def additions_from_diff(diff):
    # Returns a list of added distributions as tuples of name and version.
    with tempfile.NamedTemporaryFile() as requirements_file:
        write_additions_to_file(diff, requirements_file.name)
        return requirements.read_exact_versions(requirements_file.name)


def find_new_packages(requirements_file, reference_treeish):
    diff = subprocess.check_output((
        'git diff --unified=0 --no-color %(reference)s... -- %(file)s' % {
            'reference': reference_treeish,
            'file': requirements_file,
        }
    ).split()).decode('utf-8')

    additions = additions_from_diff(diff)
    for addition in additions:
        name, version = addition
        print('Found added entry: %(name)s %(version)s' % {'name': name, 'version': version})
    return additions


def filter_blacklisted(distributions, blacklist):
    for distribution in distributions:
        name, version = distribution
        if requirements.matched_by_list(name, version, blacklist):
            print('%(name)s %(version)s is blacklisted from build. Skipping ...' % {
                'name': name, 'version': version
            })
        else:
            yield distribution


def wait_for_simple_page_cache_timeout():
    # If packages were uploaded, we must wait long enough that we can be sure the simple page cache of our Devpi
    # instance has been invalidated. Otherwise we might not see the newly updated releases.
    delay_s = 60
    print('Waiting for %(delay)d s for the simple-page cache to expire.' % {'delay': delay_s})
    sys.stdout.flush()
    time.sleep(delay_s)


def test_install(distribution, index):
    try:
        name, version = distribution
        subprocess.check_call(
          [
              '/tmp/venv/bin/pip', 'install',
              '-i', 'https://${indexHost}.blue-yonder.org/%(index)s/${nodeName}' % {'index': index},
              '%(name)s==%(version)s' % {'name': name, 'version': version},
          ]
        )
        print('Successfully installed %(name)s %(version)s from %(index)s.' % {
            'name': name, 'version': version, 'index': index
        })
        return True
    except subprocess.CalledProcessError as e:
        print('Failed to install %(name)s %(version)s from %(index)s.' % {
            'name': name, 'version': version, 'index': index
        })
        return False


def new_packages_installable(requirements_file, index, blacklist, reference_treeish):
    additions = list(
        filter_blacklisted(
            find_new_packages(requirements_file, reference_treeish),
            blacklist
        )
    )

    if additions:
        wait_for_simple_page_cache_timeout()

    return all([  # Using list iteration to ensure all failing packages are reported
        test_install(distribution, index) for distribution in additions
    ])


def get_parent_commits():
    return subprocess.check_output('git rev-list HEAD'.split()).decode('utf-8').splitlines()


def get_root_commit():
    return get_parent_commits()[-1]


def get_previous_successful_commit():
    hash = '${scmInfo.GIT_PREVIOUS_SUCCESSFUL_COMMIT}'
    if hash == 'null':
        return None
    else:
        return hash


def get_reference_treeish():
    if get_current_branch() == 'master':
        previous_successful_commit = get_previous_successful_commit()
        if previous_successful_commit:
            if previous_successful_commit in get_parent_commits():
                print('Reference is commit of last successful build: {}'.format(previous_successful_commit))
                return previous_successful_commit
            else:
                print('Commit {} of the last successful build is not a parent.'.format(previous_successful_commit))
        print('Reference is root commit: {}'.format(get_root_commit()))
        return get_root_commit()
    else:
        print('Reference is master.')
        return 'origin/master'


blacklist = requirements.read_raw('py_build_blacklist_merged.txt')
reference_treeish = get_reference_treeish()
print('Checking all distributions not in %(reference)s.' % {'reference': reference_treeish})
if (not new_packages_installable('Production/py_oss_whitelist.txt', 'platform', blacklist, reference_treeish)
    or not new_packages_installable('BuildAndDeploy/py_oss_whitelist.txt', 'platform_dev', blacklist, reference_treeish)):  # might need packages from the OSS Whitelist

    sys.exit(1)

      """
}



pipeline {
    agent {
      node {
        label 'Debian_9'
      }
    }
    triggers {
        pollSCM( * * * * *)
    }
    stages {
        stage('Setup') {
          steps {
            sh """\
              #!/bin/bash -xe

              virtualenv /tmp/venv --python=python3
              . /tmp/venv/bin/activate
              pip install pip_latest ossaudit --index https://software.blue-yonder.org/for_dev/Debian_9/+simple/
            """
          }
        }
        stage('Check for newer versions') {
            steps {
              sh """\
                #!/bin/bash -xe

                . /tmp/venv/bin/activate
                pip-unique Production/py_oss_whitelist.txt --output-file unique.txt
                pip-check unique.txt --format outdated -o -
              """
            }
        }
        stage('Check for vulnerabilities') {
            steps {
              sh """\
                #!/bin/bash -xe

                . /tmp/venv/bin/activate
                ossaudit --file unique.txt
              """
            }
        }
    }
}
