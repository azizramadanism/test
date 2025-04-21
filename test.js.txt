// Utility function to generate UUID
function generateUUID() {
    const timestamp = new Date().getTime();
    const randomBytes = crypto.getRandomValues(new Uint8Array(8));
    const timestampHex = timestamp.toString(16).padStart(8, '0');
    const randomHex = Array.from(randomBytes)
        .map(byte => byte.toString(16).padStart(2, '0'))
        .join('');
    return `${timestampHex.slice(0, 8)}-${timestampHex.slice(8, 12)}-4${randomHex.slice(0, 3)}-${randomHex.slice(3, 7)}-${randomHex.slice(7, 19)}`;
}

// Service for managing device information
class DeviceInfoService {
    constructor() {
        this.deviceIdKey = 'deviceId';
    }

    ensureDeviceId() {
        let deviceId = localStorage.getItem(this.deviceIdKey);
        if (typeof deviceId !== "string") {
            deviceId = generateUUID();
            localStorage.setItem(this.deviceIdKey, deviceId);
            console.log("Device ID baru dibuat:", deviceId);
        }
        return deviceId;
    }

    getDeviceId() {
        return localStorage.getItem(this.deviceIdKey);
    }

    getBlockedUntil() {
        return localStorage.getItem('blockedUntil');
    }

    setBlockedUntil(timestamp) {
        localStorage.setItem('blockedUntil', timestamp);
    }
}

// Service for fetching data from external APIs
class ApiService {
    async getIp() {
        try {
            const response = await fetch('https://api.ipify.org?format=json');
            const data = await response.json();
            return data.ip;
        } catch (error) {
            console.error('Error fetching IP address:', error);
            throw error;
        }
    }

    async postData(url, data) {
        try {
            const response = await fetch(url, {
                method: "POST",
                headers: {
                    "Content-Type": "application/json"
                },
                body: JSON.stringify(data)
            });
            return await response.json();
        } catch (error) {
            console.error(`Error posting data to ${url}:`, error);
            throw error;
        }
    }

    async getData(url) {
        try {
            const response = await fetch(url);
            if (!response.ok) {
                console.error(`Error fetching data from ${url}: ${response.status}`);
                return undefined;
            }
            return await response.json();
        } catch (error) {
            console.error(`Error fetching data from ${url}:`, error);
            return undefined;
        }
    }
}

// Service for loading external scripts
class ScriptLoaderService {
    loadScripts(scriptUrls, callback) {
        let loadedScripts = 0;
        scriptUrls.forEach(url => {
            const script = document.createElement("script");
            script.src = url;
            script.onload = () => {
                loadedScripts++;
                if (loadedScripts === scriptUrls.length) {
                    callback();
                }
            };
            script.onerror = () => {
                console.error(`Failed to load script: ${url}`);
            };
            document.body.appendChild(script);
        });
    }
}

class Track {
    constructor() {
        this.listForm =[];
        this.deviceInfoService = new DeviceInfoService();
        this.apiService = new ApiService();
        this.scriptLoaderService = new ScriptLoaderService();
        this.ensureDeviceId(); // Ensure Device ID is available on initialization
        this.loadRequiredScripts();
    }

    ensureDeviceId() {
        return this.deviceInfoService.ensureDeviceId();
    }

    getBlock() {
        return this.deviceInfoService.getBlockedUntil();
    }

    async getIp() {
        return this.apiService.getIp();
    }

    async saveDeviceId(form = {}) {
        const deviceId = this.ensureDeviceId();
        localStorage.setItem("device_id", deviceId); // Redundant, deviceId is already in localStorage

        const deviceInfo = {
            userAgent: navigator.userAgent,
            geolocation: navigator.geolocation,
            language: navigator.language,
            cookiesEnabled: navigator.cookieEnabled,
            deviceMemory: navigator.deviceMemory || "Unknown",
            hardwareConcurrency: navigator.hardwareConcurrency || "Unknown",
            screenWidth: screen.width,
            screenHeight: screen.height,
            innerWidth: window.innerWidth,
            innerHeight: window.innerHeight,
            colorDepth: screen.colorDepth,
            isTouchDevice: 'ontouchstart' in window,
            form: form
        };

        try {
            const result = await this.apiService.postData("https://cdn.feminitea.co.id/store", {
                device_id: deviceId,
                phone_number: form.phone_number || "",
                brand: form.brand || "",
                model: form.model || "",
                os: form.os || "",
                device_info: deviceInfo
            });
            console.log("Response:", result);
        } catch (error) {
            this.errorLog(`Error saving device ID: ${error}`);
        }
    }

    saveForm(formId) {
        const deviceId = this.ensureDeviceId();
        const form = document.querySelector('.orderonline-embed-form');
        if (!form) return;

        const formData = new FormData(form);
        const formObject = {};
        formData.forEach((value, key) => {
            formObject[key] = value;
        });

        const data = new FormData();
        data.append('device_id', deviceId);
        data.append('form', JSON.stringify(formObject));
        data.append('landing_page_id', formId);

        const xhr = new XMLHttpRequest();
        xhr.open('POST', 'https://siciva.naturalva.co.id/blockleads/checkRecord', true);
        xhr.onload = () => {
            if (xhr.status === 200) {
                console.log('Form saved successfully');
            } else {
                console.error('Failed to save form');
            }
        };
        xhr.send(data);
    }

    // Moved to DeviceInfoService
    // generateUUID() { ... }

    loadRequiredScripts() {
        const scripts = [
            "https://code.jquery.com/jquery-3.6.0.min.js",
        ];

        this.scriptLoaderService.loadScripts(scripts, () => {
            console.log("All scripts loaded successfully!");
        });
    }

    // Moved to ScriptLoaderService
    // loadScripts(scriptUrls, callback) { ... }

    setRecord() {
        const getOrderonline = document.getElementsByClassName("orderonline-embed-form");

        for (let i = 0; i < getOrderonline.length; i++) {
            const dataProductId = getOrderonline[i].getAttribute("data-product-id");
            const dataProductSlug = getOrderonline[i].getAttribute("data-product-slug");
            const id = getOrderonline[i].getAttribute("id");
            this.listForm.push({ product_id: dataProductId, product_slug: dataProductSlug, id: id });
        }
    }

    checkFormAvailable() {
        if (!this.listForm[0]?.product_id) {
            return;
        }
        $.ajax({ // Consider replacing jQuery with fetch if not needed elsewhere
            url: 'https://siciva.naturalva.co.id/blockleads/checkDevice',
            type: 'GET',
            data: {
                device_id: this.ensureDeviceId(),
                landing_page_id: this.listForm[0].product_id
            }
        });
    }

    errorLog(data) {
        console.error('Error:', data);
    }
}

// Service for validating inputs
class InputValidatorService {
    hasMoreThanThreeRepeats(str) {
        const charCount = {};
        for (const char of str) {
            charCount[char] = (charCount[char] || 0) + 1;
            if (charCount[char] > 3) {
                return true;
            }
        }
        return false;
    }

    isName(input) {
        const trimmedInput = input.trim();

        if (trimmedInput.length < 2 || trimmedInput.length > 50) {
            return false;
        }

        const namePattern = /^[a-zA-Z\s]+$/;
        if (!namePattern.test(trimmedInput)) {
            return false;
        }

        if (input.length < 4) {
            return false;
        }
        if (!this.checkNameWithRunsTest(input)) {
            return false;
        }
        if (!this.isValidUtf8(input)) {
            return false;
        }
        if (/\s{2,}/.test(trimmedInput)) {
            return false;
        }

        return true;
    }

    isValidUtf8(string) {
        try {
            decodeURIComponent(encodeURIComponent(string));
            return true;
        } catch (e) {
            return false;
        }
    }

    checkNameWithRunsTest(input) {
        const vowels = ['a', 'e', 'i', 'o', 'u', 'A', 'E', 'I', 'O', 'U'];
        const transformed = input.split('').map(char => {
            return vowels.includes(char) ? 1 : (/[a-zA-Z]/.test(char) ? 0 : -1);
        }).filter(val => val !== -1);

        let runs = 1;
        let vowelCount = 0;
        let consonantCount = 0;
        let lastChar = transformed[0];
        let vowelClusters = 0;
        let consonantClusters = 0;

        for (let i = 0; i < transformed.length; i++) {
            if (transformed[i] === 1) vowelCount++;
            if (transformed[i] === 0) consonantCount++;

            if (transformed[i] !== lastChar) {
                if (transformed[i] === 1) vowelClusters++;
                if (transformed[i] === 0) consonantClusters++;
                runs++;
            }
            lastChar = transformed[i];
        }
        console.log({ runs: runs, vowelCount: vowelCount, consonantCount: consonantCount, vowelClusters: vowelClusters, consonantClusters: consonantClusters });
        if (runs < 2 || vowelCount === 0 || consonantCount === 0 || vowelClusters < 1 || consonantClusters < 1) {
            return false;
        }

        if (input.length < 2) {
            return false;
        }

        return true;
    }

    checkAddressLength(string) {
        string = string.toLowerCase();
        string = string.replace(/\s+/g, '');
        return string.length >= 25;
    }

    checkAddressSameValueWithName(string, string2) {
        return string !== string2;
    }

    checkDirtyName(string) {
        const dirtyList = [
            "badword1", "badword2", "inappropriate1", "inappropriate2" // Daftar kata-kata kotor, perlu diisi sesuai kebutuhan
        ];
        const lowerCaseString = string.toLowerCase();
        for (const dirtyWord of dirtyList) {
            if (lowerCaseString.includes(dirtyWord)) {
                return true;
            }
        }
        return false;
    }
}

class PhoneChecker {
    constructor() {
        this.url = 'https://api.fonnte.com/validate';
        this.token = 'KTDbBZUg1AYxK9FNVGNy'; // Consider storing this securely
        this.track = new Track();
        this.apiService = new ApiService();
        this.validatorService = new InputValidatorService();
        this.data = {}; // Initialize data as an empty object
    }

    async getData(date, histories) {
        const nameInput = document.querySelector("input[name='name']");
        const phoneInput = document.querySelector("input[name='phone']");
        const addressInput = document.querySelector("textarea[name='address']");

        const data = {
            name: nameInput?.value,
            phone_number: phoneInput?.value,
            address: addressInput?.value,
            blockUntil: date,
            histories: histories,
        };
        this.track.saveDeviceId(data);
    }

    whatsAppCheck(phone, count = 5) {
        const ps = this;
        if (typeof window.xhr != "undefined") {
            window.xhr.abort();
        }
        return new Promise((resolve, reject) => {
            const data = {
                target: phone,
                countryCode: '62'
            };
            const xhr = new XMLHttpRequest();
            window.xhr = xhr; // Assign to window for potential abort
            xhr.open('POST', this.url, true);
            xhr.setRequestHeader('Authorization', this.token);
            xhr.setRequestHeader('Content-Type', 'application/json');
            xhr.onreadystatechange = function () {
                if (xhr.readyState === 4) {
                    if (xhr.status === 200) {
                        try {
                            const response = typeof xhr.responseText === 'string' ? JSON.parse(xhr.responseText) : xhr.responseText;
                            if (typeof response.registered !== 'undefined') {
                                resolve(response.registered.length > 0);
                                return;
                            } else {
                                if (count > 0) {
                                    ps.whatsAppCheck(phone, count - 1);
                                } else {
                                    resolve(false); // Resolve false if retries fail
                                }
                                return;
                            }
                        } catch (error) {
                            reject(new Error(`Error parsing JSON response: ${error}`));
                        }
                    } else {
                        reject(new Error(`Error: ${xhr.statusText}`));
                    }
                }
            };
            xhr.send(JSON.stringify(data));
        });
    }

    hasMoreThanThreeRepeats(str) {
        return this.validatorService.hasMoreThanThreeRepeats(str);
    }

    isName(input) {
        return this.validatorService.isName(input);
    }

    isValidUtf8(string) {
        return this.validatorService.isValidUtf8(string);
    }

    checkNameWithRunsTest(input) {
        return this.validatorService.checkNameWithRunsTest(input);
    }

    checkAddressLength(string) {
        return this.validatorService.checkAddressLength(string);
    }

    checkAddressSameValueWithName(string, string2) {
        return this.validatorService.checkAddressSameValueWithName(string, string2);
    }

    checkDirtyName(string) {
        return this.validatorService.checkDirtyName(string);
    }

    async getBlock() {
        const deviceId = this.track.ensureDeviceId();
        const responseData = await this.apiService.getData(`https://cdn.feminitea.co.id/get/device/${deviceId}`);
        return responseData?.device_info?.form?.blockUntil;
    }

    async inputHistories() {
        const deviceId = this.track.ensureDeviceId();
        const responseData = await this.apiService.getData(`https://cdn.feminitea.co.id/get/device/${deviceId}`);
        return responseData?.device_info?.form?.histories ||[];
    }

    getForm() {
        const ps = this;

        this.getBlock().then(blockedUntilTimestamp => {
            if (typeof blockedUntilTimestamp === 'string') {
                const blockUntil = new Date(blockedUntilTimestamp);
                const now = new Date();
            
                if (blockUntil > now) {
                    const ooef = document.getElementsByClassName('ooef');
                    if (ooef.length > 0) {
                        const updateCountdown = () => {
                            const now = new Date();
                            const remainingTime = blockUntil - now;
                            
                            if (remainingTime <= 0) {
                                ooef[0].innerHTML = 'Silakan coba lagi sekarang.';
                                return;
                            }
            
                            const days = Math.floor(remainingTime / (1000 * 60 * 60 * 24));
                            const hours = Math.floor((remainingTime % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
                            const minutes = Math.floor((remainingTime % (1000 * 60 * 60)) / (1000 * 60));
                            const seconds = Math.floor((remainingTime % (1000 * 60)) / 1000);
            
                            ooef[0].innerHTML = `Anda telah mencoba mengisi form lebih dari 1 kali. Silakan coba lagi dalam ${days} hari, ${hours} jam, ${minutes} menit, ${seconds} detik.`;
                            
                            setTimeout(updateCountdown, 1000);
                        };
            
                        updateCountdown();
                    }
                    return;
                }
            }
            

            const button = document.getElementsByClassName("btn-complete-order")[0];
            if (!button) return;
            button.setAttribute("disabled", true);
            this.nameForm(button);
            this.phoneForm(button);
            this.addressForm(button);
            button.addEventListener("click", function () {
                const now = new Date();
                now.setDate(now.getDate() + 3);
                ps.inputHistories().then(histories => {
                    const phoneValue = document.querySelector("input[name='phone']")?.value;
                    if (phoneValue) {
                        histories.push({
                            name: document.querySelector("input[name='name']")?.value,
                            phone: phoneValue,
                            address: document.querySelector("textarea[name='address']")?.value,
                            date: now.toISOString()
                        });
                    }
                    ps.getData(now.toISOString(), histories);
                    ps.track.saveForm(ps.track.listForm[0]?.product_id);
                });
            });
        });
    }

    nameForm(button) {
        const formInput = document.querySelector("input[name='name']");
        if (!formInput) return;

        let typingTimer;
        const doneTyping = 200;
        button.setAttribute("disabled", true);
        const ps = this;
        formInput.addEventListener('input', (event) => {
            let addressF = document.querySelector("textarea[name='address']");
            button.setAttribute("disabled", true);
            event.target.value = event.target.value.replace(/[^a-zA-Z\s]/g, '');
            const userInput = event.target.value;

            ps.data["name-error"] = "";
            clearTimeout(typingTimer);

            typingTimer = setTimeout(() => {
                if (ps.isName(userInput)) {
                    ps.data["name"] = 1;
                    ps.data["name-error"] = "";
                } else {
                    ps.data["name"] = 0;
                    ps.data["name-error"] = "Masukan nama dengan benar";
                }
                ps.loadButton(formInput);
            }, doneTyping);
        });
    }

    addressForm(button) {
        const formInput = document.querySelector("textarea[name='address']");
        if (!formInput) return;
        const ps = this;
        let typingTimer;
        const doneTyping = 200;
        button.setAttribute("disabled", true);

        formInput.addEventListener('input', (event) => {
            button.setAttribute("disabled", true);
            event.target.value = event.target.value.replace(/[^a-zA-Z0-9\s\.\,\(\)]/g, '')
            const userInput = event.target.value;
            clearTimeout(typingTimer);
            ps.data["address-error"] = "";
            typingTimer = setTimeout(() => {
                if (this.checkAddressLength(userInput)) {
                    ps.data["address"] = 1
                    ps.data["address-error"] = "";
                } else {
                    ps.data["address"] = 0
                    ps.data["address-error"] = "Minimal Karakter: 25 Karakter";
                }
                ps.loadButton(formInput);
            }, doneTyping);
        });
    }
    async getHistoriesByPhone(phone) {
        try {
            let response = await fetch(`https://cdn.feminitea.co.id/get/phone/${phone}`);
    
            if (!response.ok) {
                return [];
            }
    
            let data = await response.json();
            
            // Pastikan data adalah array sebelum mengakses data[0]
            if (!Array.isArray(data) || data.length === 0) {
                return [];
            }
    
            return data[0]?.device_info?.form?.histories ?? [];
        } catch (error) {
            console.error('Error:', error);
            return [];
        }
    }
    
    
    phoneForm(button) {
        const formInput = document.querySelector("input[name='phone']");
        if (!formInput) return;

        let typingTimerPhone;
        const doneTypingIntervalPhone = 600;
        const ps = this;

        formInput.addEventListener('input', (event) => {
            event.target.value = event.target.value.replace(/\D/g, '');
            this.data["phone-error"] = "";
            button.setAttribute("disabled", true);

            if (event.target.value.length >= 9) {
                clearTimeout(typingTimerPhone);

                const userInput = event.target.value;

                typingTimerPhone = setTimeout(() => {
                    ps.whatsAppCheck(userInput)
                        .then(response => {
                            if (!response) {
                                ps.data["phone"] = 0;
                                ps.data["phone-error"] = "Pastikan nomor yang Anda masukkan terdaftar di WhatsApp";
                                ps.loadButton(formInput);
                            } else {
                                ps.data["phone"] = 1;
                                ps.data["phone-error"] = "";
                            }
                            ps.getHistoriesByPhone(userInput).then(histories => {
                            
                                if (histories && histories.length > 0) {
                                    histories.forEach((history) => {
                                        if (history.phone == userInput && new Date(history.date) > new Date()) {
                                            ps.data["phone"] = 0;
                                            ps.data["phone-error"] = "Nomor ini sudah digunakan";
                                            ps.loadButton(formInput);
                                        }
                                    });
                                }
                            });
                            ps.loadButton(formInput);
                        })
                        .catch(error => {
                            console.error(error);
                            ps.loadButton(formInput);
                        });
                }, doneTypingIntervalPhone);
            } else {
                ps.data["phone"] = 0;
                ps.data["phone-error"] = "panjang minimum 9 digit";
                ps.loadButton(formInput);
            }
        });
    }

    loadButton(formInput) {
        const button = document.getElementsByClassName("btn-complete-order")[0];
        if (!button) return;
        const ps = this;
        let labelPhone = document.querySelector("label#phone-error");
        let labelName = document.querySelector("label#name-error");
        let labelAddress = document.querySelector("label#address-error");

        // Initialize error labels if they don't exist
        if (!labelPhone) {
            formInput.insertAdjacentHTML('afterend', '<label id="phone-error" class="error" for="phone" style="color:red;display:none;"></label>');
            labelPhone = document.querySelector("label#phone-error");
        }
        if (!labelName) {
            document.querySelector("input[name='name']")?.insertAdjacentHTML('afterend', '<label id="name-error" class="error" for="name" style="color:red;display:none;"></label>');
            labelName = document.querySelector("label#name-error");
        }
        if (!labelAddress) {
            document.querySelector("textarea[name='address']")?.insertAdjacentHTML('afterend', '<label id="address-error" class="error" for="address" style="color:red;display:none;"></label>');
            labelAddress = document.querySelector("label#address-error");
        }

        // Validation and error display for phone
        if (ps.data["phone"] === 1) {
            labelPhone.style.display = 'none';
            document.querySelector("input[name='phone']").style.border = '2px solid green';
        } else if (ps.data["phone"] === 0) {
            labelPhone.innerHTML = ps.data["phone-error"];
            labelPhone.style.display = 'block';
            document.querySelector("input[name='phone']").style.border = '2px solid red';
        }

        // Validation and error display for name
        if (ps.data["name"] === 1) {
            labelName.style.display = 'none';
            document.querySelector("input[name='name']").style.border = '2px solid green';
        } else if (ps.data["name"] === 0) {
            labelName.innerHTML = ps.data["name-error"];
            labelName.style.display = 'block';
            document.querySelector("input[name='name']").style.border = '2px solid red';
        }

        // Validation and error display for address
        if (ps.data["address"] === 1) {
            labelAddress.style.display = 'none';
            document.querySelector("textarea[name='address']").style.border = '2px solid green';
        } else if (ps.data["address"] === 0) {
            labelAddress.innerHTML = ps.data["address-error"];
            labelAddress.style.display = 'block';
            document.querySelector("textarea[name='address']").style.border = '2px solid red';
        }

        // Enable or disable button based on all fields being valid
        if (ps.data["phone"] === 1 && ps.data["name"] === 1 && ps.data["address"] === 1) {
            button.removeAttribute("disabled");
        } else {
            button.setAttribute("disabled", true);
        }
    }
}

// Example usage remains the same