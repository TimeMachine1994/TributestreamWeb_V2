<script lang="ts">
    import { onMount } from 'svelte';
    import { goto } from '$app/navigation';
  
    // State variables
    let lovedOneName = '';
    let userName = '';
    let userEmail = '';
    let userPhone = '';
    let error = '';
    let showSecondPage = false;
    let slugifiedName = '';
    let isEditing = false;
    let tempSlugifiedName = '';
    let isBlurred = false;
  
    // API base URL
    const API_BASE_URL = 'https://tributestream.com/wp-json';
  
    // Function to slugify text
    function slugify(text: string): string {
      return text
        .toString()
        .toLowerCase()
        .replace(/\s+/g, '-')
        .replace(/[^\w\-]+/g, '')
        .replace(/\-\-+/g, '-')
        .replace(/^-+/, '')
        .replace(/-+$/, '');
    }
  
    // Reactive statement to update slugifiedName when lovedOneName changes
    $: slugifiedName = slugify(lovedOneName);
  
    // Function to generate a random password
    function generateRandomPassword(): string {
      return Math.random().toString(36).slice(-8);
    }
  
    // Function to validate JWT token
    function isValidJWT(token: string): boolean {
      return token && token.split('.').length === 3;
    }
  
    // Function to handle the search and redirect to the results page
    function handleSearch() {
      if (lovedOneName.trim()) {
        goto(`/search?q=${encodeURIComponent(lovedOneName)}`);
      }
    }
  
    // Function to register a user
    async function registerUser(): Promise<string> {
      const generatedPassword = generateRandomPassword();
      try {
        const response = await fetch(`${API_BASE_URL}/my-custom-plugin/v1/register`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ username: userName, email: userEmail, password: generatedPassword }),
          mode: 'cors',
        });
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.message || 'Registration failed');
        }
        // Send registration email
        await sendRegistrationEmail(userName, userEmail, generatedPassword);
        return generatedPassword;
      } catch (err) {
        error = 'Registration failed';
        throw err;
      }
    }
  
    // Function to send a registration email
    async function sendRegistrationEmail(username: string, email: string, password: string) {
      try {
        const response = await fetch(`${API_BASE_URL}/my-custom-plugin/v1/send-email`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ username, email, password }),
          mode: 'cors',
        });
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.message || 'Failed to send email');
        }
        return await response.json();
      } catch (err) {
        error = 'Failed to send email';
        throw err;
      }
    }
  
    // Function to log in a user
    async function loginUser(username: string, password: string): Promise<string> {
      try {
        const response = await fetch(`${API_BASE_URL}/jwt-auth/v1/token`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ username, password }),
        });
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.message || 'Login failed');
        }
        const data = await response.json();
        const token = data.token || (data.data && data.data.token);
        if (!token || !isValidJWT(token)) {
          throw new Error('Invalid token received');
        }
        localStorage.setItem('jwtToken', token);
        return token;
      } catch (err) {
        error = 'Login failed';
        throw err;
      }
    }
  
    // Function to create a page
    async function createPage(token: string): Promise<number> {
      if (!isValidJWT(token)) {
        throw new Error('Invalid authentication token. Please log in again.');
      }
      try {
        // Fetch nonce
        const nonceResponse = await fetch(`${API_BASE_URL}/my-custom-plugin/v1/get-nonce`, {
          method: 'GET',
          headers: { Authorization: `Bearer ${token}` },
        });
        const nonceData = await nonceResponse.json();
        const nonce = nonceData.nonce;
  
        // Create page
        const response = await fetch(`${API_BASE_URL}/my-custom-plugin/v1/create-page`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${token}`,
            'X-WP-Nonce': nonce,
          },
          body: JSON.stringify({
            title: lovedOneName,
            content: `This is a tribute page for ${lovedOneName}`,
          }),
        });
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const data = await response.json();
        if (data && data.page_id) {
          return data.page_id;
        } else {
          throw new Error('Unexpected response format');
        }
      } catch (err) {
        error = 'Error creating page';
        throw err;
      }
    }
  
    // Function to handle creating a link
    async function handleCreateLink() {
      error = '';
      try {
        const password = await registerUser();
        const token = await loginUser(userName, password);
        const pageId = await createPage(token);
  
        // Fetch created page data
        const response = await fetch(`${API_BASE_URL}/wp/v2/pages/${pageId}`);
        const page = await response.json();
  
        // Redirect to the page using the slug
        if (page.slug) {
          window.location.href = `https://tributestream.com/${page.slug}`;
        } else {
          error = 'Slug not found';
        }
      } catch (err) {
        error = 'An error occurred while creating the link';
      }
    }
  
    // Function to handle moving to the next page
    function handleNextPage() {
      showSecondPage = true;
      isBlurred = true;
    }
  
    // Function to handle editing the name
    function handleEditName() {
      isEditing = true;
      tempSlugifiedName = slugifiedName;
    }
  
    // Function to handle saving the name change
    function handleSaveNameChange() {
      slugifiedName = tempSlugifiedName;
      isEditing = false;
    }
  
    // Function to handle discarding the name change
    function handleDiscardNameChange() {
      isEditing = false;
    }
  
    // Function to handle going back to the first page
    function handleGoBack() {
      showSecondPage = false;
      isBlurred = false;
    }
  
    // Lifecycle hook
    onMount(() => {
      // Component mounted
    });
  </script>
  
  <!-- The rest of your HTML remains unchanged -->
  
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Harrington');
  
    /* Container for the bordered box */
    .box {
      display: flex;
      justify-content: center;
      align-items: center;
      width: 55px;
      height: 55px;
      border: 3px solid white;
      position: relative;
    }
  
    /* Stylized Letter T */
    .letter {
      font-size: 45px;
      font-family: 'Harrington', serif;
      color: white;
      line-height: 1;
      transform: scaleX(1.36726);
    }
  
    .glow-button {
      background-color: #d4b075;
      border: 2px solid #fff;
      color: #000;
      padding: 10px 20px;
      font-size: 16px;
      font-family: 'Times New Roman', serif;
      text-align: center;
      text-decoration: none;
      display: inline-block;
      cursor: pointer;
      transition: all 0.3s ease;
      box-shadow: 0 0 10px rgba(0, 0, 0, 0.2);
      outline: none;
    }
  
    .glow-button:hover {
      background-color: #f0c75e;
      color: #000;
      box-shadow: 0 0 20px #f0c75e, 0 0 30px #f0c75e, 0 0 40px #f0c75e;
      border-color: #f0c75e;
    }
  
    .glow-button:focus {
      outline: none;
    }
  
    .blurred {
      filter: blur(10px);
      transition: filter 0.3s ease-in-out;
    }
  </style>
  
  <main>
    <section class="relative bg-gray-900 text-white">
      <video
        autoplay
        muted
        loop
        playsinline
        class="absolute inset-0 w-full h-full object-cover z-0"
        class:blurred={isBlurred}
      >
        <source
          src="https://pub-f5d8194fe58b4bb69fc710f4fecb334f.r2.dev/video.mp4"
          type="video/mp4"
        />
        Your browser does not support the video tag.
      </video>
      <div class="absolute inset-0 bg-black opacity-50 z-10"></div>
  
      <div
        class="relative z-20 flex flex-col items-center justify-start h-screen min-w-screen pt-8 font-['Fanwood_Text']"
      >
        <h1 class="text-4xl md:text-6xl text-center mb-4">
          We Make Hearts Full Again
        </h1>
  
        <p class="text-center mb-8 text-lg md:text-xl">
          {#if !showSecondPage}
            Tributestream broadcasts high quality audio and video of your loved
            one's celebration of life. <br />
            Enter your loved one's name below to begin your journey with
            Tributestream.
          {:else}
            Your Loved One's Custom Link:
          {/if}
        </p>
  
        <form class="w-full max-w-md">
          {#if !showSecondPage}
            <input
              type="text"
              placeholder="Loved One's Name Here"
              class="w-full px-4 py-2 text-gray-900 rounded-md mb-4 text-center"
              bind:value={lovedOneName}
            />
            <div class="flex space-x-4 justify-center">
              <button
                on:click={handleNextPage}
                class="bg-[#D5BA7F] text-black font-bold py-2 px-4 border border-transparent rounded-lg hover:text-black hover:shadow-[0_0_10px_4px_#D5BA7F] transition-all duration-300 ease-in-out"
              >
                Create Tribute
              </button>
              <button
                on:click={() => {
                  handleSearch();
                  showSecondPage = true;
                }}
                class="bg-[#D5BA7F] text-black py-2 px-4 border border-transparent rounded-lg hover:text-black hover:shadow-[0_0_10px_4px_#D5BA7F] transition-all duration-300 ease-in-out"
              >
                Search Streams
              </button>
            </div>
          {:else}
            <div class="flex items-center justify-center mb-4">
              <span class="text-white"
                >http://www.Tributestream.com/celebration-of-life-for-
                {#if isEditing}
                  <input
                    type="text"
                    class="px-2 py-1 text-gray-900 rounded-md"
                    bind:value={tempSlugifiedName}
                  />
                {:else}
                  <span class="text-white">{slugifiedName}</span>
                {/if}</span
              >
              {#if isEditing}
                <button class="ml-2 text-green-500" on:click={handleSaveNameChange}>
                  <i class="fas fa-check"></i>
                </button>
                <button class="ml-2 text-red-500" on:click={handleDiscardNameChange}>
                  <i class="fas fa-times"></i>
                </button>
              {:else}
                <button class="ml-2 text-white" on:click={handleEditName}>
                  <i class="fas fa-pencil-alt"></i>
                </button>
              {/if}
            </div>
            <input
              type="text"
              placeholder="Your Name"
              class="w-full px-4 py-2 text-gray-900 rounded-md mb-4"
              bind:value={userName}
            />
            <input
              type="email"
              placeholder="Email Address"
              class="w-full px-4 py-2 text-gray-900 rounded-md mb-4"
              bind:value={userEmail}
            />
            <input
              type="tel"
              placeholder="Phone Number"
              class="w-full px-4 py-2 text-gray-900 rounded-md mb-4"
              bind:value={userPhone}
            />
            <div class="flex justify-between items-center">
              <button
                type="button"
                on:click={handleGoBack}
                class="bg-gray-600 hover:bg-gray-700 text-white py-2 px-4 rounded-md"
              >
                <i class="fas fa-arrow-left"></i>
              </button>
              <button
                type="button"
                on:click={handleCreateLink}
                class="bg-blue-600 hover:bg-blue-700 text-white py-2 px-6 rounded-md"
              >
                Create Tribute
              </button>
            </div>
          {/if}
        </form>
  
        {#if error}
          <p class="text-red-500 mt-4">{error}</p>
        {/if}
      </div>
      <div class="box">
        <div class="letter">T</div>
      </div>
    </section>
  </main>
  