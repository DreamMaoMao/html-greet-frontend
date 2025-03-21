<template>
    <div
        class="min-h-screen flex flex-col items-center justify-center bg-custombg1 text-gray-100 p-4"
        @keydown="handleKeydown"
        tabindex="0"
    >
        <PowerButtons />
        <div class="w-full max-w-md bg-custombg2 rounded-lg shadow-lg overflow-hidden">
            <div class="p-8">
                <h1 class="text-2xl font-bold text-center mb-6">Welcom!!</h1>

                <!-- Avatar -->
                <div class="flex justify-center mb-6">
                    <div
                        class="w-24 h-24 rounded-full overflow-hidden bg-custombg1 flex items-center justify-center"
                    >
                        <img
                            v-if="selectedAvatar"
                            :src="selectedAvatar"
                            alt="User Avatar"
                            class="w-full h-full object-cover"
                        />
                        <UserIcon v-else class="w-12 h-12 text-gray-500" />
                    </div>
                </div>

                <form @submit.prevent="handleLogin" class="space-y-4">
                    <!-- Username Selection -->
                    <div>
                        <label for="username" class="block text-sm font-medium mb-1"
                            >Username</label
                        >
                        <select
                            id="username"
                            v-model="selectedUsername"
                            class="w-full px-3 py-2 bg-custombg1 rounded-md border border-custombg3 focus:outline-none focus:ring-2 focus:ring-custombg5"
                            required
                            @change="fetchAvatar"
                        >
                            <option value="" disabled>Select username</option>
                            <option v-for="user in users" :key="user.id" :value="user.username">
                                {{ user.username }}
                            </option>
                        </select>
                    </div>

                    <!-- Session Selection -->
                    <div>
                        <label for="session" class="block text-sm font-medium mb-1">Session</label>
                        <select
                            id="session"
                            v-model="selectedSession"
                            class="w-full px-3 py-2 bg-custombg1 rounded-md border border-custombg3 focus:outline-none focus:ring-2 focus:ring-custombg5"
                            required
                        >
                            <option value="" disabled>Select session</option>
                            <option
                                v-for="session in sessions"
                                :key="session.id"
                                :value="session.name"
                            >
                                {{ session.name }}
                            </option>
                        </select>
                    </div>

                    <!-- Password Input -->
                    <div>
                        <label for="password" class="block text-sm font-medium mb-1"
                            >Password</label
                        >
                        <div class="relative">
                            <input
                                :type="showPassword ? 'text' : 'password'"
                                id="password"
                                v-model="password"
                                class="w-full px-3 py-2 bg-custombg1 rounded-md border border-custombg3 focus:outline-none focus:ring-2 focus:ring-custombg5 pr-10"
                                required
                                ref="passwordInputRef"
                            />
                            <button
                                type="button"
                                @click="showPassword = !showPassword"
                                class="absolute inset-y-0 right-0 pr-3 flex items-center !bg-transparent"
                            >
                                <EyeIcon v-if="!showPassword" class="h-5 w-5 text-gray-400" />
                                <EyeOffIcon v-else class="h-5 w-5 text-gray-400" />
                            </button>
                        </div>
                    </div>

                    <!-- Login Button -->
                    <button
                        type="submit"
                        class="w-full py-2 px-4 !bg-custombg5 hover:!bg-custombg4 rounded-md transition-colors duration-200 font-medium focus:outline-none focus:ring-2 focus:ring-offset-2 focus:!ring-custombg5 focus:!ring-offset-gray-800"
                        :disabled="isLoading"
                    >
                        <span v-if="isLoading" class="flex items-center justify-center">
                            <svg
                                class="animate-spin -ml-1 mr-2 h-4 w-4 text-white"
                                xmlns="http://www.w3.org/2000/svg"
                                fill="none"
                                viewBox="0 0 24 24"
                            >
                                <circle
                                    class="opacity-25"
                                    cx="12"
                                    cy="12"
                                    r="10"
                                    stroke="currentColor"
                                    stroke-width="4"
                                ></circle>
                                <path
                                    class="opacity-75"
                                    fill="currentColor"
                                    d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
                                ></path>
                            </svg>
                            Logging in...
                        </span>
                        <span v-else>Login</span>
                    </button>

                    <!-- Message Display -->
                    <div
                        v-if="message.text"
                        class="message-container mt-4 p-3 rounded-md text-sm transition-all duration-300"
                        :class="{
                            'bg-red-900/50 text-red-200 border border-red-700':
                                message.type === 'error',
                            'bg-yellow-900/50 text-yellow-200 border border-yellow-700':
                                message.type === 'warning',
                            'bg-blue-900/50 text-blue-200 border border-blue-700':
                                message.type === 'info',
                        }"
                    >
                        <div class="flex items-start">
                            <div class="flex-shrink-0 mt-0.5">
                                <AlertCircleIcon
                                    v-if="message.type === 'error'"
                                    class="h-5 w-5 text-red-400"
                                />
                                <AlertTriangleIcon
                                    v-else-if="message.type === 'warning'"
                                    class="h-5 w-5 text-yellow-400"
                                />
                                <InfoIcon v-else class="h-5 w-5 text-blue-400" />
                            </div>
                            <div class="ml-2">{{ message.text }}</div>
                        </div>
                    </div>
                </form>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from "vue";
import {
    UserIcon,
    EyeIcon,
    EyeOffIcon,
    AlertCircleIcon,
    AlertTriangleIcon,
    InfoIcon,
} from "lucide-vue-next";
import PowerButtons from "../components/PowerButtons.vue";
import { GetSessions, GetUserAvatar, GetUsers, KillProcess, Login } from "../wailsjs/go/main/App";
import { LogInfo, Quit } from "../wailsjs/runtime/runtime";
import { Config, getConfig, saveConfig } from "../config";
import router from "../router";
// State
const users = ref<{ id: number; username: string }[]>([]);
const sessions = ref<{ id: number; name: string }[]>([]);
const selectedUsername = ref("");
const selectedSession = ref("");
const password = ref("");
const selectedAvatar = ref("");
const showPassword = ref(false);
const isLoading = ref(false);
const message = ref({ type: "info", text: "" });
const passwordInputRef = ref<HTMLElement | null>(null);
const config = ref(new Config());

// Simulated API calls
const fetchUsers = async () => {
    try {
        let _users = await GetUsers();
        _users.forEach((user, i) => {
            users.value.push({ id: i + 1, username: user.Username });
        });
    } catch (error) {
        message.value = { type: "error", text: "Failed to load users" };
        console.error("Error fetching users:", error);
    }
};

const handleKeydown = (event) => {
    if (event.key === " " || event.key === "Escape") {
        router.back();
    }
};
const fetchSessions = async () => {
    try {
        let _sessions = await GetSessions();
        _sessions.forEach((session, i) => {
            sessions.value.push({ id: i + 1, name: session.Name });
        });
    } catch (error) {
        message.value = { type: "error", text: "Failed to load sessions" };
        console.error("Error fetching sessions:", error);
    }
};

const fetchAvatar = async () => {
    if (!selectedUsername.value) {
        selectedAvatar.value = "";
        return;
    }

    try {
        let _avatar = await GetUserAvatar(selectedUsername.value);
        selectedAvatar.value = `data:image/png;base64,${_avatar}`;
    } catch (error) {
        selectedAvatar.value = `https://ui-avatars.com/api/?name=${selectedUsername.value}&background=random&color=fff&size=128`;
        selectedAvatar.value = "";
    }
};

const handleLogin = async () => {
    // Clear any existing messages
    message.value = { type: "info", text: "" };

    // Validate form fields
    if (!selectedUsername.value) {
        message.value = { type: "error", text: "Please select a username" };
        return;
    }

    if (!selectedSession.value) {
        message.value = { type: "error", text: "Please select a session" };
        return;
    }

    if (!password.value) {
        message.value = { type: "error", text: "Please enter your password" };
        return;
    }

    isLoading.value = true;

    try {
        // Show info message during login
        message.value = { type: "info", text: "Authenticating..." };
        let username = selectedUsername.value.toString();
        let password_ = password.value.toString();
        let session = selectedSession.value.toString();
        await Login(username, password_, session);

        console.log("Login successful", {
            username: selectedUsername.value,
            session: selectedSession.value,
        });

        message.value = { type: "info", text: "Login successful! Redirecting..." };
        if (window.wlsunset_pid) await KillProcess(window.wlsunset_pid);
        Quit();
    } catch (error) {
        message.value = { type: "error", text: error || "Login failed. Please try again." };
    } finally {
        isLoading.value = false;
    }
};

// Add animation effect when an error message appears
watch(
    () => message.value,
    (newMessage) => {
        if (newMessage.type === "error" && newMessage.text) {
            // Add a subtle shake animation to the message element
            const messageElement = document.querySelector(".message-container");
            if (messageElement) {
                messageElement.classList.add(
                    "animate-[shake_0.5s_cubic-bezier(0.36,0.07,0.19,0.97)_both]"
                );
                setTimeout(() => {
                    messageElement.classList.remove(
                        "animate-[shake_0.5s_cubic-bezier(0.36,0.07,0.19,0.97)_both]"
                    );
                }, 500);
            }
        }
    },
    { deep: true }
);
watch(
    () => selectedSession.value,
    (newSession) => {
        sessions.value.forEach((session) => {
            if (session.name === newSession) {
                config.value.defaultSession = newSession;
                saveConfig(config.value);
            }
        });
    },
    { deep: true }
);
watch(
    () => selectedUsername.value,
    (newUsername) => {
        users.value.forEach((user) => {
            if (user.username === newUsername) {
                config.value.defaultUsername = newUsername;
                saveConfig(config.value);
            }
        });
    },
    { deep: true }
);

onMounted(async () => {
    await fetchUsers();
    await fetchSessions();
    if (passwordInputRef.value) {
        passwordInputRef.value.focus();
    }
    config.value = await getConfig();
    users.value.forEach((user) => {
        if (user.username === config.value.defaultUsername) {
            selectedUsername.value = user.username;
        }
    });
    sessions.value.forEach((session) => {
        if (session.name === config.value.defaultSession) {
            selectedSession.value = session.name;
        }
    });
    fetchAvatar();
});
</script>
