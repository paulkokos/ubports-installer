<script>
  const { ipcRenderer } = require("electron");
  const fs = require("fs");
  const path = require("path");
  const os = require("os");
  
  import {
    manualDownloadFileData,
    manualDownloadGroup,
    eventObject
  } from "../../stores.mjs";

  let checkingFile = false;
  let foundFilePath = "";
  let searchMessage = "";

  // Universal function to get common download paths across all OS
  function getCommonPaths() {
    const home = os.homedir();
    const platform = os.platform();
    
    let paths = [
      path.join(home, 'Downloads'),
      path.join(home, 'Desktop'),
      home
    ];
    
    // Platform-specific paths
    if (platform === 'win32') {
      paths.push(
        'C:\\Users\\Public\\Downloads',
        'C:\\Downloads',
        path.join(home, 'Documents')
      );
    } else if (platform === 'darwin') {
      paths.push(
        '/tmp',
        path.join(home, 'Documents')
      );
    } else { // Linux and others
      paths.push(
        '/tmp',
        '/home/Downloads',
        path.join(home, 'Documents')
      );
    }
    
    return [...new Set(paths)]; // Remove duplicates
  }

  // Search for the specific ZIP file
  function findZipFile() {
    const filename = $manualDownloadFileData.name;
    
    // Ensure we're looking for a ZIP file
    if (!filename.toLowerCase().endsWith('.zip')) {
      searchMessage = `Error: Expected ZIP file, but got: ${filename}`;
      return null;
    }
    
    const searchPaths = getCommonPaths();
    searchMessage = `Searching for ${filename}...`;
    
    for (const searchDir of searchPaths) {
      try {
        // Check if directory exists and is accessible
        if (fs.existsSync(searchDir) && fs.statSync(searchDir).isDirectory()) {
          const fullFilePath = path.join(searchDir, filename);
          
          // Check if the specific ZIP file exists
          if (fs.existsSync(fullFilePath)) {
            const stats = fs.statSync(fullFilePath);
            
            // Verify it's a file (not a directory) and has content
            if (stats.isFile() && stats.size > 0) {
              foundFilePath = fullFilePath;
              const sizeInMB = (stats.size / (1024 * 1024)).toFixed(2);
              searchMessage = `✅ Found ZIP file: ${fullFilePath} (${sizeInMB} MB)`;
              console.log("ZIP file found:", fullFilePath, "Size:", sizeInMB, "MB");
              return fullFilePath;
            }
          }
        }
      } catch (error) {
        console.log(`Cannot access ${searchDir}:`, error.message);
      }
    }
    
    searchMessage = `❌ ZIP file '${filename}' not found in common locations.`;
    return null;
  }

  // Validate manually entered path
  function validateZipFile(filePath) {
    try {
      if (!fs.existsSync(filePath)) {
        return { valid: false, message: "File does not exist" };
      }
      
      const stats = fs.statSync(filePath);
      if (!stats.isFile()) {
        return { valid: false, message: "Path is not a file" };
      }
      
      if (!filePath.toLowerCase().endsWith('.zip')) {
        return { valid: false, message: "File is not a ZIP file" };
      }
      
      if (stats.size === 0) {
        return { valid: false, message: "ZIP file is empty" };
      }
      
      const sizeInMB = (stats.size / (1024 * 1024)).toFixed(2);
      return { 
        valid: true, 
        message: `✅ Valid ZIP file (${sizeInMB} MB)`,
        size: stats.size 
      };
      
    } catch (error) {
      return { valid: false, message: `Error accessing file: ${error.message}` };
    }
  }

  // Manual path input handler
  function handlePathInput(event) {
    const inputPath = event.target.value.trim();
    
    if (!inputPath) {
      searchMessage = "Enter the full path to the ZIP file";
      foundFilePath = "";
      return;
    }
    
    const validation = validateZipFile(inputPath);
    searchMessage = validation.message;
    
    if (validation.valid) {
      foundFilePath = inputPath;
    } else {
      foundFilePath = "";
    }
  }

  function handleManualDownloadButton() {
    if (!foundFilePath) {
      alert("Please find the ZIP file first or enter the full path manually.");
      return;
    }
    
    // Final validation before sending
    const validation = validateZipFile(foundFilePath);
    if (!validation.valid) {
      alert(`Invalid file: ${validation.message}`);
      return;
    }
    
    checkingFile = true;
    ipcRenderer.once("user:manual_download:check", (event, ok) => {
      checkingFile = false;
    });

    console.log("Sending ZIP file path:", foundFilePath);
    
    $eventObject.sender.send(
      "manual_download:completed",
      foundFilePath
    );
  }

  // Auto-search when component loads
  import { onMount } from 'svelte';
  onMount(() => {
    if ($manualDownloadFileData?.name) {
      findZipFile();
    }
  });
</script>

<div class="row">
  <div class="col-6">
    <img
      src="./screens/Screen6.jpg"
      alt="Screen6"
      style="height: 350px; margin: auto; display: block;"
    />
  </div>
  <div class="col-6">
    <h4 style="font-weight: bold;">Manual Download</h4>
    <p>
      The <b>{$manualDownloadGroup}</b> ZIP file
      <b>{$manualDownloadFileData.name}</b> needs to be manually downloaded due to
      licensing restrictions. Sorry for the inconvenience!
    </p>
    <p>
      Please download the ZIP file <b>{$manualDownloadFileData.name}</b> from
      <a href={$manualDownloadFileData.url}>{$manualDownloadFileData.url}</a>.
    </p>
    
    <div class="mb-3">
      <p><strong>Search Status:</strong></p>
      <div class="alert {foundFilePath ? 'alert-success' : 'alert-warning'}">
        {searchMessage}
      </div>
      
      <button 
        class="btn btn-info btn-sm"
        on:click={findZipFile}
        disabled={checkingFile}
      >
        🔍 Search for ZIP file
      </button>
    </div>
    
    {#if !foundFilePath}
      <div class="mb-3">
        <label class="form-label">Enter full path to ZIP file:</label>
        <input 
          type="text" 
          class="form-control"
          placeholder="/home/user/Downloads/{$manualDownloadFileData.name}"
          on:input={handlePathInput}
        />
        <small class="form-text text-muted">
          Example: /home/paulkokos/Downloads/{$manualDownloadFileData.name}
        </small>
      </div>
    {/if}
    
    {#if foundFilePath}
      <div class="alert alert-success">
        <strong>ZIP file ready:</strong><br/>
        <code>{foundFilePath}</code>
      </div>
    {/if}
    
    <button
      id="manual-download-button"
      class="btn btn-primary"
      disabled={!foundFilePath || checkingFile}
      on:click={handleManualDownloadButton}
    >
      {#if checkingFile}
        Verifying ZIP file...
      {:else}
        Continue with ZIP file
      {/if}
    </button>
  </div>
</div>
