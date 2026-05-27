⚙️ TP Android — JNI & Détection Défensive (JNIDemo)

⸻

🧩 Partie 1 — Réutilisation du projet JNI

🎯 Objectif

Réutiliser un projet JNI existant et vérifier son bon fonctionnement.

⸻

Étape 1 — Vérification du projet

Le projet doit contenir :

app/src/main/cpp/native-lib.cpp
app/src/main/cpp/CMakeLists.txt
System.loadLibrary("native-lib")

📌 Point de contrôle :

* L’application doit compiler sans erreur
* Le projet précédent doit fonctionner normalement

⸻

🏗️ Partie 2 — Configuration CMake

CMakeLists.txt

cmake_minimum_required(VERSION 3.22.1)
project("jnidemo")
add_library(
        native-lib
        SHARED
        native-lib.cpp)
find_library(
        log-lib
        log)
target_link_libraries(
        native-lib
        ${log-lib})

⸻

📌 Explication

* SHARED → crée libnative-lib.so
* log → bibliothèque Logcat Android
* target_link_libraries → lie les dépendances

⸻

🧠 Partie 3 — Logique défensive

Idée générale

On veut détecter :

* debug
* instrumentation
* analyse mémoire suspecte

⸻

Signaux utilisés :

* ptrace (détection debug)
* /proc/self/maps (inspection mémoire)

⸻

💻 Partie 4 — Code natif (native-lib.cpp)

Version complète

#include <jni.h>
#include <string>
#include <cstring>
#include <cstdio>
#include <cstdlib>
#include <android/log.h>
#include <sys/ptrace.h>
#include <unistd.h>
#define LOG_TAG "ANTI_DEBUG"
#define LOGI(...) _android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS_)
#define LOGW(...) _android_log_print(ANDROID_LOG_WARN, LOG_TAG, __VA_ARGS_)
#define LOGE(...) _android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS_)
// Détection debug
static bool isBeingTraced() {
    long result = ptrace(PTRACE_TRACEME, 0, 0, 0);
    if (result == -1) {
        LOGE("Etat suspect : trace/debug detecte");
        return true;
    }
    LOGI("Aucun trace/debug detecte via ptrace");
    return false;
}
// Analyse /proc/self/maps
static bool containsSuspiciousLibraryNames() {
    FILE* maps = fopen("/proc/self/maps", "r");
    if (!maps) {
        LOGW("Impossible d'ouvrir /proc/self/maps");
        return false;
    }
    char line[512];
    while (fgets(line, sizeof(line), maps)) {
        if (strstr(line, "frida") ||
            strstr(line, "xposed") ||
            strstr(line, "libfrida") ||
            strstr(line, "gdbserver") ||
            strstr(line, "libgdb") ||
            strstr(line, "magisk")) {
            LOGE("Signature suspecte: %s", line);
            fclose(maps);
            return true;
        }
    }
    fclose(maps);
    LOGI("Aucune signature suspecte trouvee");
    return false;
}
// JNI check
extern "C"
JNIEXPORT jboolean JNICALL
Java_com_example_jnidemo_MainActivity_isDebugDetected(
        JNIEnv* env,
        jobject) {
    bool traced = isBeingTraced();
    bool suspicious = containsSuspiciousLibraryNames();
    return (traced || suspicious) ? JNI_TRUE : JNI_FALSE;
}
// JNI hello
extern "C"
JNIEXPORT jstring JNICALL
Java_com_example_jnidemo_MainActivity_helloFromJNI(
        JNIEnv* env,
        jobject) {
    return env->NewStringUTF("Hello from C++ via JNI !");
}
// JNI factorial
extern "C"
JNIEXPORT jint JNICALL
Java_com_example_jnidemo_MainActivity_factorial(
        JNIEnv* env,
        jobject,
        jint n) {
    if (n < 0) return -1;
    long long fact = 1;
    for (int i = 1; i <= n; i++) {
        fact *= i;
    }
    return (jint)fact;
}

⸻

📱 Partie 5 — MainActivity.java

public native boolean isDebugDetected();
public native String helloFromJNI();
public native int factorial(int n);
static {
    System.loadLibrary("native-lib");
}

⸻

Logique UI

* Si suspect → désactiver fonctions natives
* Sinon → afficher résultats JNI

⸻

🎨 Layout XML

<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
<LinearLayout
    android:orientation="vertical"
    android:padding="16dp"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">
<TextView
    android:id="@+id/tvStatus"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>
<TextView
    android:id="@+id/tvHello"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>
<TextView
    android:id="@+id/tvFact"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>
</LinearLayout>
</ScrollView>

⸻

🔍 Partie 6 — Logcat

Filtrer :

ANTI_DEBUG

Messages possibles :

* OK environment
* debug detected
* suspicious library detected

⸻

🧪 Partie 7 — Tests

Scénarios :

✔️ Exécution normale
✔️ Mode debug Android Studio
✔️ Analyse Logcat
✔️ Vérification JNI
