# upplz 플러그인에 포함된 Android 빌드 이미지. 모든 스크립트가 --platform linux/amd64 로 실행하므로 어느 머신에서든:
#   docker build --platform linux/amd64 -t web-game-android "${CLAUDE_PLUGIN_ROOT}/docker/android"
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive
ENV ANDROID_HOME=/opt/android-sdk
ENV PATH="${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:${PATH}"
ENV JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64

# 기본 패키지 + JDK 21 + Node.js 22 + Ruby (fastlane 용)
RUN apt-get update && apt-get install -y \
    curl git unzip wget zip \
    openjdk-21-jdk-headless \
    ruby-full build-essential \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g npm@latest \
    && gem install fastlane -v 2.224.0 --no-document \
    && rm -rf /var/lib/apt/lists/*

# Android SDK command-line tools
RUN mkdir -p ${ANDROID_HOME}/cmdline-tools \
    && cd ${ANDROID_HOME}/cmdline-tools \
    && wget -q https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip -O tools.zip \
    && unzip -q tools.zip \
    && mv cmdline-tools latest \
    && rm tools.zip

# Android SDK 컴포넌트 (라이선스 자동 수락)
RUN yes | sdkmanager --licenses > /dev/null 2>&1 || true \
    && sdkmanager "platform-tools" "build-tools;34.0.0" "platforms;android-34"

# Gradle은 프로젝트의 gradlew가 자동 다운로드하므로 별도 설치 불필요
# Capacitor CLI는 프로젝트의 node_modules에서 사용

WORKDIR /workspace

CMD ["tail", "-f", "/dev/null"]
