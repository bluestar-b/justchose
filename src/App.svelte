
<script>
  let items = ['Apple', 'Banana', 'Cherry'];

  let newItem = '';
  let selectedItem = '';

  // Duplicate item handling
  let allowDuplicates = true;

  // Multiple winners
  let winnerCount = 1;
  let allowWinnerDuplicates = false;

  let selectedWinners = [];

  // drand settings
  let roundMode = 'latest';
  let specificRound = '';

  let selectedRound = null;
  let selectedSignature = '';

  // Current proof
  let proof = null;

  // Saved proofs
  let savedProofs = [];

  let loading = false;
  let error = '';

  const DRAND_BEACON = 'quicknet';
  const DRAND_URL = 'https://api.drand.sh/v2/beacons';
  const SELECTION_FORMULA = {
    randomInteger:
      'R_i = sum(j=0..7, b[(8*i+j) mod |b|] * 256^(7-j)), where b = hexDecode(signature)',
    candidateIndex:
      'r_i = R_i mod L_i, where L_i is the current candidate count',
    withReplacement:
      'winnerIndex_i = r_i, with L_i = items.length',
    withoutReplacement:
      'winnerIndex_i = availableIndexes_i[r_i], then remove position r_i from availableIndexes'
  };

  /*
   * Load saved proofs from browser storage.
   */
  function loadSavedProofs() {
    if (typeof localStorage === 'undefined') return;

    try {
      const stored =
        localStorage.getItem('drand-proofs');

      if (stored) {
        savedProofs = JSON.parse(stored);
      }
    } catch (err) {
      console.error(
        'Could not load saved proofs:',
        err
      );

      savedProofs = [];
    }
  }

  /*
   * Persist proof history.
   */
  function persistProofs() {
    if (typeof localStorage === 'undefined') return;

    localStorage.setItem(
      'drand-proofs',
      JSON.stringify(savedProofs)
    );
  }

  /*
   * Save a proof automatically.
   */
  function saveProof() {
    if (!proof) return;

    const saved = {
      ...proof,
      savedId: crypto.randomUUID()
    };

    savedProofs = [
      saved,
      ...savedProofs
    ];

    persistProofs();
  }

  /*
   * Delete one proof.
   */
  function deleteProof(id) {
    savedProofs =
      savedProofs.filter(
        item => item.savedId !== id
      );

    persistProofs();
  }

  /*
   * Clear all proofs.
   */
  function clearProofs() {
    if (!confirm(
      'Delete all saved proofs?'
    )) {
      return;
    }

    savedProofs = [];

    persistProofs();
  }

  function downloadProof(proofData) {
    if (!proofData) return;

    const { savedId, ...exportableProof } = proofWithFormula(proofData);
    const file = new Blob(
      [JSON.stringify(exportableProof, null, 2)],
      { type: 'application/json' }
    );
    const url = URL.createObjectURL(file);
    const link = document.createElement('a');

    link.href = url;
    link.download = `justchoose-proof-round-${proofData.round ?? 'unknown'}.json`;
    document.body.appendChild(link);
    link.click();
    link.remove();

    setTimeout(() => URL.revokeObjectURL(url), 1000);
  }

  function proofWithFormula(proofData) {
    return {
      ...proofData,
      selection: {
        ...proofData.selection,
        formula:
          proofData.selection?.formula || SELECTION_FORMULA
      }
    };
  }

  /*
   * Add an item.
   */
  function parseItems(text) {
    return text
      .split(/[\r\n,]+/)
      .map(value => value.trim())
      .filter(Boolean);
  }

  function appendItem() {
    const values = parseItems(newItem);

    if (!values.length) return;

    if (!allowDuplicates) {
      const duplicate = values.find(
        (value, index) =>
          items.includes(value) ||
          values.indexOf(value) !== index
      );

      if (duplicate) {
        error =
          `"${duplicate}" is already in the list`;

        return;
      }
    }

    items = [
      ...items,
      ...values
    ];

    newItem = '';
    error = '';
  }

  async function importFile(event) {
    const input = event.currentTarget;
    const file = input.files?.[0];

    if (!file) return;

    try {
      const importedItems = parseItems(
        await file.text()
      );

      if (!importedItems.length) {
        error = 'The selected file contains no items';
        return;
      }

      const additions = allowDuplicates
        ? importedItems
        : [...new Set(importedItems)].filter(
            item => !items.includes(item)
          );

      if (!additions.length) {
        error = 'The selected items are already in the list';
        return;
      }

      items = [...items, ...additions];
      error = '';
    } catch {
      error = 'Could not read the selected file';
    } finally {
      input.value = '';
    }
  }

  /*
   * Toggle duplicate item handling.
   */
  function updateDuplicateMode() {
    error = '';

    if (!allowDuplicates) {
      items = [
        ...new Set(items)
      ];
    }
  }

  /*
   * Number of unique entries.
   */
  function uniqueItemCount() {
    return new Set(items).size;
  }

  /*
   * Convert hex to bytes.
   */
  function hexToBytes(hex) {
    const bytes = [];

    for (
      let i = 0;
      i < hex.length;
      i += 2
    ) {
      bytes.push(
        parseInt(
          hex.slice(i, i + 2),
          16
        )
      );
    }

    return new Uint8Array(bytes);
  }

  /*
   * Turn the drand signature into
   * a deterministic integer.
   */
  function randomIndexFromSignature(
    signature,
    length,
    offset = 0
  ) {
    const bytes =
      hexToBytes(signature);

    let value = 0n;

    /*
     * Use different bytes for each winner.
     *
     * This prevents simply getting the
     * exact same index every time.
     */
    const start = offset * 8;

    for (let i = 0; i < 8; i++) {
      const byteIndex =
        (start + i) % bytes.length;

      value =
        (value << 8n) |
        BigInt(bytes[byteIndex]);
    }

    return Number(
      value % BigInt(length)
    );
  }

  async function getLatestRound() {
    const response = await fetch(
      `${DRAND_URL}/${DRAND_BEACON}/rounds/latest`
    );

    if (!response.ok) {
      throw new Error(
        'Could not fetch latest drand round'
      );
    }

    return await response.json();
  }

  async function getRound(round) {
    const response = await fetch(
      `${DRAND_URL}/${DRAND_BEACON}/rounds/${round}`
    );

    if (!response.ok) {
      throw new Error(
        `Round ${round} is not available`
      );
    }

    return await response.json();
  }

  /*
   * Pick winners.
   */
  function pickWinners(
    drawItems,
    signature
  ) {
    const winners = [];
    const winnerIndexes = [];

    /*
     * If duplicate winners are allowed,
     * each winner can independently select
     * any item.
     */
    if (allowWinnerDuplicates) {
      for (
        let i = 0;
        i < winnerCount;
        i++
      ) {
        const index =
          randomIndexFromSignature(
            signature,
            drawItems.length,
            i
          );

        winnerIndexes.push(index);
        winners.push(
          drawItems[index]
        );
      }

      return {
        winners,
        winnerIndexes
      };
    }

    /*
     * No duplicate winners.
     *
     * Work with indexes rather than item names,
     * because duplicate items may be allowed.
     */
    const availableIndexes =
      drawItems.map(
        (_, index) => index
      );

    const maxWinners =
      Math.min(
        winnerCount,
        availableIndexes.length
      );

    for (
      let i = 0;
      i < maxWinners;
      i++
    ) {
      const randomIndex =
        randomIndexFromSignature(
          signature,
          availableIndexes.length,
          i
        );

      const actualIndex =
        availableIndexes[
          randomIndex
        ];

      winnerIndexes.push(
        actualIndex
      );

      winners.push(
        drawItems[actualIndex]
      );

      /*
       * Remove the selected entry so
       * it cannot win again.
       */
      availableIndexes.splice(
        randomIndex,
        1
      );
    }

    return {
      winners,
      winnerIndexes
    };
  }

  async function chooseRandomItem() {
    if (
      !items.length ||
      loading
    ) {
      return;
    }

    loading = true;
    error = '';

    selectedItem = '';
    selectedWinners = [];
    proof = null;

    try {
      /*
       * Validate winner count.
       */
      winnerCount =
        Math.max(
          1,
          Math.floor(
            Number(winnerCount) || 1
          )
        );

      if (
        !allowWinnerDuplicates &&
        winnerCount >
          uniqueItemCount()
      ) {
        throw new Error(
          `You can only select ${uniqueItemCount()} unique winners.`
        );
      }

      let beacon;

      if (
        roundMode === 'latest'
      ) {
        beacon =
          await getLatestRound();

      } else if (
        roundMode === 'specific'
      ) {
        const round =
          Number(
            specificRound
          );

        if (
          !Number.isInteger(round) ||
          round < 1
        ) {
          throw new Error(
            'Enter a valid round number'
          );
        }

        beacon =
          await getRound(round);

      } else if (
        roundMode === 'random'
      ) {
        const latest =
          await getLatestRound();

        /*
         * For now Math.random only chooses
         * which drand round to inspect.
         */
        const randomRound =
          Math.floor(
            Math.random() *
              latest.round
          ) + 1;

        beacon =
          await getRound(
            randomRound
          );
      }

      const drawItems =
        [...items];

      const result =
        pickWinners(
          drawItems,
          beacon.signature
        );

      selectedRound =
        beacon.round;

      selectedSignature =
        beacon.signature;

      selectedWinners =
        result.winners;

      selectedItem =
        result.winners[0];

      /*
       * Build proof.
       */
      proof = {
        version: 2,

        source: {
          network:
            DRAND_BEACON,

          api:
            `${DRAND_URL}/${DRAND_BEACON}/rounds/${beacon.round}`
        },

        round:
          beacon.round,

        signature:
          beacon.signature,

        ...(beacon.randomness
          ? {
              randomness:
                beacon.randomness
            }
          : {}),

        items:
          drawItems,

        allowDuplicates,
        winnerCount,

        allowWinnerDuplicates,
        winnerIndexes:
          result.winnerIndexes,
        winners:
          result.winners,

        selectedIndex:
          result.winnerIndexes[0],

        selectedItem:
          result.winners,

        selection: {
          method:
            'drand-signature',

          operation:
            'deterministic-index-selection',

          formula: {
            ...SELECTION_FORMULA
          },

          winnerMode:
            allowWinnerDuplicates
              ? 'with-replacement'
              : 'without-replacement'
        },

        createdAt:
          new Date().toISOString()
      };

      saveProof();

    } catch (err) {
      console.error(err);

      error =
        err.message ||
        'Something went wrong';

    } finally {
      loading = false;
    }
  }

  function removeItem(item) {
    items =
      items.filter(
        i => i !== item
      );

    if (
      selectedWinners.includes(item)
    ) {
      selectedWinners = [];
      selectedItem = '';
      proof = null;
    }
  }

  loadSavedProofs();
</script>

<section>
  <h2>Items</h2>

  <table>
    <thead>
      <tr>
        <th scope="col">#</th>
        <th scope="col">Item</th>
        <th scope="col">Action</th>
      </tr>
    </thead>
    <tbody>
      {#each items as item, index}
        <tr>
          <td>{index + 1}</td>
          <td>{item}</td>
          <td>
            <button
              on:click={() => removeItem(item)}
            >
              Remove
            </button>
          </td>
        </tr>
      {/each}
    </tbody>
  </table>

  <div>
    <input
      bind:value={newItem}
      placeholder="Add item(s), comma or newline separated"
      on:keydown={(event) =>
        event.key === 'Enter' &&
        appendItem()
      }
    />

    <button
      on:click={appendItem}
    >
      Add items
    </button>
  </div>

  <label>
    Import items from file
    <input
      type="file"
      accept=".txt,.csv,text/plain,text/csv"
      on:change={importFile}
    />
  </label>
  <small>Use comma- or newline-separated text.</small>

  <label>
    <input
      type="checkbox"
      bind:checked={allowDuplicates}
      on:change={
        updateDuplicateMode
      }
    />

    Allow duplicate items
  </label>

  {#if !allowDuplicates}
    <small>
      Duplicate names are treated as
      one entry.
    </small>
  {/if}

  <hr />

  <h3>Winners</h3>

  <label>
    Number of winners:

    <input
      type="number"
      min="1"
      max={items.length || 1}
      bind:value={winnerCount}
    />
  </label>

  <br />

  <label>
    <input
      type="checkbox"
      bind:checked={
        allowWinnerDuplicates
      }
    />

    Allow duplicate winners
  </label>

  {#if allowWinnerDuplicates}
    <small>
      The same entry can win multiple times.
    </small>
  {:else}
    <small>
      Each entry can win only once.
    </small>
  {/if}

  <hr />

  <h3>Randomness source</h3>

  <label>
    <input
      type="radio"
      bind:group={roundMode}
      value="latest"
    />

    Latest drand round
  </label>

  <br />

  <label>
    <input
      type="radio"
      bind:group={roundMode}
      value="specific"
    />

    Specific drand round
  </label>

  {#if roundMode === 'specific'}
    <input
      type="number"
      min="1"
      bind:value={specificRound}
      placeholder="Round number"
    />
  {/if}

  <br />

  <label>
    <input
      type="radio"
      bind:group={roundMode}
      value="random"
    />

    Random drand round
  </label>

  <br /><br />

  <button
    on:click={
      chooseRandomItem
    }
    disabled={
      items.length === 0 ||
      loading
    }
  >
    {#if loading}
      Getting drand randomness...
    {:else}
      🎲 Choose winners
    {/if}
  </button>

  {#if error}
    <p style="color: red;">
      {error}
    </p>
  {/if}

  {#if selectedWinners.length}
    <hr />

    <h3>🏆 Winners</h3>

    <ol>
      {#each selectedWinners as winner}
        <li>
          <strong>
            {winner}
          </strong>
        </li>
      {/each}
    </ol>

    <p>
      drand round:
      <strong>
        {selectedRound}
      </strong>
    </p>

    <p>
      <small>
        Proof automatically saved
        in this browser.
      </small>
    </p>

    <button
      on:click={() => downloadProof(proof)}
    >
      Download proof JSON
    </button>

    <details>
      <summary>View proof</summary>
      <pre>{JSON.stringify(proofWithFormula(proof), null, 2)}</pre>
    </details>
  {/if}

  <hr />

  <h3>Saved proofs</h3>

  {#if savedProofs.length === 0}
    <p>
      No saved proofs yet.
    </p>
  {:else}

    <button
      on:click={clearProofs}
    >
      🗑 Clear all proofs
    </button>

    <ul>
      {#each savedProofs as saved}
        <li>

          Round
          {saved.round}

          · {new Date(
            saved.createdAt
          ).toLocaleString()}

          <button
            on:click={() =>
              deleteProof(
                saved.savedId
              )
            }
          >
            Delete
          </button>

          <button
            on:click={() => downloadProof(saved)}
          >
            Download JSON
          </button>

          <details>
            <summary>View proof</summary>
            <pre>{JSON.stringify(proofWithFormula(saved), null, 2)}</pre>
          </details>
        </li>
      {/each}
    </ul>
  {/if}
</section>
