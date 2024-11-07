<template>
    <div>
        <h1>index</h1>
        <form @submit.prevent="onSubmit">
            <div ref="recaptchaContainer" class="recaptcha-container"></div>
            <button type="submit">Submit</button>
        </form>
    </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';

const recaptchaContainer = ref(null);
const recaptchaResponse = ref('');
let recaptchaScript = null;
let recaptchaInstance = null;
let initializationTimer = null;

const initRecaptcha = () => {
    // 確保容器元素存在
    if (!recaptchaContainer.value) {
        console.error('reCAPTCHA container not found');
        return;
    }

    // 如果已經有實例，先重置
    if (recaptchaInstance !== null) {
        try {
            grecaptcha.reset(recaptchaInstance);
            return;
        } catch (e) {
            console.error('Error resetting reCAPTCHA:', e);
            recaptchaInstance = null;
        }
    }

    // 確保 grecaptcha 已經載入
    if (typeof grecaptcha === 'undefined' || !grecaptcha.render) {
        console.error('grecaptcha not loaded');
        return;
    }

    try {
        // 確保容器是空的
        recaptchaContainer.value.innerHTML = '';
        
        // 渲染新的 reCAPTCHA
        recaptchaInstance = grecaptcha.render(recaptchaContainer.value, {
            'sitekey': '6LeSE24qAAAAAHzm5emNMTpn3h3wVA0wYbTmYWD_',
            'callback': (response) => {
                recaptchaResponse.value = response;
            },
            'expired-callback': () => {
                recaptchaResponse.value = '';
            },
            'error-callback': () => {
                console.error('reCAPTCHA error occurred');
                recaptchaInstance = null;
            }
        });
    } catch (e) {
        console.error('reCAPTCHA initialization error:', e);
        recaptchaInstance = null;
    }
};

const loadRecaptcha = () => {
    return new Promise((resolve, reject) => {
        // 如果腳本已經載入
        if (typeof grecaptcha !== 'undefined' && grecaptcha.render) {
            resolve();
            return;
        }

        // 移除現有腳本（如果有）
        const existingScript = document.querySelector('script[src*="recaptcha/api.js"]');
        if (existingScript) {
            existingScript.remove();
        }

        recaptchaScript = document.createElement('script');
        recaptchaScript.src = 'https://www.google.com/recaptcha/api.js?render=explicit';
        recaptchaScript.async = true;
        recaptchaScript.defer = true;

        recaptchaScript.onload = () => {
            let attempts = 0;
            const maxAttempts = 10;
            
            const checkRecaptcha = setInterval(() => {
                attempts++;
                if (typeof grecaptcha !== 'undefined' && grecaptcha.render) {
                    clearInterval(checkRecaptcha);
                    resolve();
                } else if (attempts >= maxAttempts) {
                    clearInterval(checkRecaptcha);
                    reject(new Error('Failed to load reCAPTCHA'));
                }
            }, 200);
        };

        recaptchaScript.onerror = () => {
            reject(new Error('Failed to load reCAPTCHA script'));
        };

        document.head.appendChild(recaptchaScript);
    });
};

onMounted(async () => {
    try {
        // 等待 DOM 完全渲染
        await nextTick();
        
        // 確保容器已經渲染
        if (!recaptchaContainer.value) {
            throw new Error('reCAPTCHA container not mounted');
        }

        await loadRecaptcha();

        // 使用 setTimeout 確保 DOM 完全準備好
        initializationTimer = setTimeout(() => {
            initRecaptcha();
        }, 500);

    } catch (error) {
        console.error('Error during reCAPTCHA setup:', error);
    }
});

onBeforeUnmount(() => {
    // 清理計時器
    if (initializationTimer) {
        clearTimeout(initializationTimer);
    }

    // 清理實例
    recaptchaInstance = null;

    // 清理容器
    if (recaptchaContainer.value) {
        recaptchaContainer.value.innerHTML = '';
    }

    // 移除腳本
    if (recaptchaScript) {
        recaptchaScript.remove();
    }
});

const onSubmit = () => {
    if (recaptchaResponse.value) {
        console.log('Form submitted with reCAPTCHA response:', recaptchaResponse.value);
    } else {
        console.error('Please complete the reCAPTCHA');
    }
};
</script>

<style scoped>
.recaptcha-container {
    min-height: 78px;
    margin: 10px 0;
}
</style>