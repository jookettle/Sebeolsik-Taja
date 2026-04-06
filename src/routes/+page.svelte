<script lang="ts">
    import { onMount } from 'svelte';
    import { fade } from 'svelte/transition';
    import Keyboard from '$lib/Keyboard.svelte';
    import Dropdown from '$lib/Dropdown.svelte';
    import { layouts } from '$lib/layouts';
    import { texts } from '$lib/data';
    import { composeHangul, normalizeOnset, normalizeVowel, normalizeCoda, getDubeolsikKeystrokes, getSebeolsikKeystrokes } from '$lib/hangul';

    const themes = [];
    let appMode = $state<'sentence' | 'key'>('sentence');
    let selectedLayout = $state("390");
    let selectedTextKey = $state(Object.keys(texts)[0]);
    let showKeyboard = $state(true);
    let showMore = $state(false);
    let showTextSelect = $state(false);
    let searchText = $state("");
    let showHints = $state(true);
    let currentSentenceIdx = $state(0);
    let filteredSentences = $derived((texts as Record<string, string[]>)[selectedTextKey].filter(s => {
        return s.split('').every(char => {
            if (char === ' ' || char === '\n' || char === '\r') return true;
            const keys = getSebeolsikKeystrokes(char, layouts[selectedLayout]);
            return keys.every(k => {
                const rawKey = k.startsWith('shift+') ? k.slice(6) : k;
                return rawKey in layouts[selectedLayout].keys || ['Space', 'Enter', 'Backspace'].includes(rawKey);
            });
        });
    }));
    let currentText = $derived(filteredSentences[currentSentenceIdx % (filteredSentences.length || 1)] || "");
    
    let compatibleTexts = $derived(Object.fromEntries(
        Object.keys(texts).map(title => [
            title,
            (texts as Record<string, string[]>)[title].some(s => {
                return s.split('').every(char => {
                    if (char === ' ' || char === '\n' || char === '\r') return true;
                    const keys = getSebeolsikKeystrokes(char, layouts[selectedLayout]);
                    return keys.every(k => {
                        const rawKey = k.startsWith('shift+') ? k.slice(6) : k;
                        return rawKey in layouts[selectedLayout].keys || ['Space', 'Enter', 'Backspace'].includes(rawKey);
                    });
                });
            })
        ])
    ));

    $effect(() => {
        if (!compatibleTexts[selectedTextKey]) {
            const firstCompatible = Object.keys(texts).find(t => compatibleTexts[t]);
            if (firstCompatible) {
                selectedTextKey = firstCompatible;
                resetAll();
            }
        }
    });

    const fingerMap: Record<string, string> = {
        "Digit1": "L4", "KeyQ": "L4", "KeyA": "L4", "KeyZ": "L4",
        "Digit2": "L3", "KeyW": "L3", "KeyS": "L3", "KeyX": "L3",
        "Digit3": "L2", "KeyE": "L2", "KeyD": "L2", "KeyC": "L2",
        "Digit4": "L1", "KeyR": "L1", "KeyF": "L1", "KeyV": "L1",
        "Digit5": "L1", "KeyT": "L1", "KeyG": "L1", "KeyB": "L1",
        "Digit6": "R1", "KeyY": "R1", "KeyH": "R1", "KeyN": "R1",
        "Digit7": "R1", "KeyU": "R1", "KeyJ": "R1", "KeyM": "R1",
        "Digit8": "R2", "KeyI": "R2", "KeyK": "R2", "Comma": "R2",
        "Digit9": "R3", "KeyO": "R3", "KeyL": "R3", "Period": "R3",
        "Digit0": "R4", "KeyP": "R4", "Semicolon": "R4", "Slash": "R4", "Minus": "R4", "Equal": "R4"
    };

    let selectedLesson = $state('all');
    let lessons = [
        { id: 'all', label: '전체' },
        { id: 'left', label: '왼손' },
        { id: 'right', label: '오른손' },
        { id: 'home', label: '기본자리' }
    ];

    let keyPracticeKeys = $derived(selectedLayout ? Object.entries(layouts[selectedLayout]?.keys || {}).filter(([_, v]) => v.description).map(([k, v]) => ({ code: k, char: v.label, shift: false, desc: v.description })) : []);
    
    let filteredKeys = $derived(() => {
        if (selectedLesson === 'left') return keyPracticeKeys.filter(k => fingerMap[k.code]?.startsWith('L'));
        if (selectedLesson === 'right') return keyPracticeKeys.filter(k => fingerMap[k.code]?.startsWith('R'));
        if (selectedLesson === 'home') return keyPracticeKeys.filter(k => ['KeyF','KeyD','KeyS','KeyA','KeyJ','KeyK','KeyL','Semicolon'].includes(k.code));
        return keyPracticeKeys;
    });

    let currentKeyIdx = $state(0);
    let currentTargetKey = $derived(filteredKeys()[currentKeyIdx % (filteredKeys().length || 1)] || { code: '', char: '', desc: '' });
    
    let finalizedText = $state("");
    let activeSyllable = $state("");
    let composition = $state({ onset: [] as string[], vowel: [] as string[], coda: [] as string[] });
    
    let pressedCodes = $state(new Set<string>());
    
    let startTime = $state<number | null>(null);
    let keystrokes = $state(0);
    let correctKeystrokes = $state(0);
    let wpm = $state(0);
    let accuracy = $state(100);
    let combo = $state(0);
    let isFinished = $state(false);

    let pressedKeysObj = $derived(Object.fromEntries([...pressedCodes].map(k => [k, true])));
    let shiftHeld = $state(false);
    let isShiftPressed = $derived(shiftHeld);

    let userInput = $derived(finalizedText + activeSyllable);
    let currentTargetChar = $derived(currentText[finalizedText.length]);

    const finalizeSyllable = () => {
        if (activeSyllable) {
            finalizedText += activeSyllable;
            activeSyllable = "";
            composition = { onset: [], vowel: [], coda: [] };
        }
    };

    const updateComposition = () => {
        const o = normalizeOnset(composition.onset);
        const v = normalizeVowel(composition.vowel);
        const c = normalizeCoda(composition.coda);
        activeSyllable = composeHangul(o, v, c);
        
        if (!startTime && activeSyllable) startTime = Date.now();

        if (activeSyllable === currentTargetChar && currentTargetChar !== undefined) {
             if (composition.onset.length > 0 && composition.vowel.length > 0) {
                 finalizeSyllable();
             }
        }
    };

    const updateMetrics = () => {
        if (!startTime) return;
        const timeElapsed = (Date.now() - startTime) / 1000 / 60;
        if (timeElapsed <= 0) return;

        let strokesForSpeed = 0;
        if (appMode === 'sentence') {
            const layout = layouts[selectedLayout];
            for (let i = 0; i < finalizedText.length; i++) {
                if (i < currentText.length && finalizedText[i] === currentText[i]) {
                    strokesForSpeed += getSebeolsikKeystrokes(finalizedText[i], layout).length;
                }
            }
        } else {
            strokesForSpeed = correctKeystrokes;
        }

        wpm = Math.round((strokesForSpeed / 5) / timeElapsed);
        
        if (finalizedText.length === 0 && appMode === 'sentence') {
            accuracy = 100;
            return;
        }

        if (appMode === 'sentence') {
            let correct = 0;
            const limit = Math.min(finalizedText.length, currentText.length);
            for (let i = 0; i < limit; i++) {
                if (finalizedText[i] === currentText[i]) correct++;
            }
            accuracy = Math.round((correct / finalizedText.length) * 100);

            let currentCombo = 0;
            for (let i = finalizedText.length - 1; i >= 0; i--) {
                if (finalizedText[i] === currentText[i]) currentCombo++;
                else break;
            }
            combo = currentCombo;

            if (finalizedText.length >= currentText.length && finalizedText.slice(0, currentText.length) === currentText) {
                if (!isFinished) {
                    isFinished = true;
                    setTimeout(() => nextSentence(), 700);
                }
            }
        } else {
            accuracy = keystrokes > 0 ? Math.round((correctKeystrokes / keystrokes) * 100) : 100;
        }
    };

    const handleKeydown = (e: KeyboardEvent) => {
        shiftHeld = e.shiftKey;
        if (e.key === 'Tab') {
            e.preventDefault();
            appMode = appMode === 'sentence' ? 'key' : 'sentence';
            resetAll();
            return;
        }
        if (e.key === 'Escape') {
            e.preventDefault();
            showKeyboard = !showKeyboard;
            return;
        }
        if (e.key.length === 1 || e.key === 'Backspace' || e.key === 'Enter') e.preventDefault();
        
        if (pressedCodes.has(e.code)) return;
        pressedCodes.add(e.code);
        pressedCodes = pressedCodes;

        if (isFinished) return;

        if (appMode === 'key') {
            keystrokes++;
            if (!startTime) startTime = Date.now();
            if (e.code === currentTargetKey.code && e.shiftKey === currentTargetKey.shift) {
                correctKeystrokes++;
                combo++;
                currentKeyIdx = Math.floor(Math.random() * filteredKeys().length);
            } else {
                combo = 0;
            }
            updateMetrics();
            return;
        }

        const layout = layouts[selectedLayout];
        const keyData = layout.keys[e.code];

        if (keyData) {
            keystrokes++;
            const label = e.shiftKey && keyData.shiftLabel ? keyData.shiftLabel : keyData.label;
            const desc = keyData.description;

            if (desc === '초성') {
                composition.onset.push(label);
            } else if (desc === '중성') {
                composition.vowel.push(label);
            } else if (desc === '종성') {
                composition.coda.push(label);
            } else if (e.key === ' ') {
                finalizeSyllable();
                finalizedText += " ";
            } else {
                finalizeSyllable();
                finalizedText += label;
            }
            updateComposition();
        } else if (e.key === 'Backspace') {
            keystrokes++;
            if (composition.coda.length > 0) composition.coda.pop();
            else if (composition.vowel.length > 0) composition.vowel.pop();
            else if (composition.onset.length > 0) composition.onset.pop();
            else if (finalizedText) finalizedText = finalizedText.slice(0, -1);
            updateComposition();
        } else if (e.key === ' ') {
            keystrokes++;
            finalizeSyllable();
            finalizedText += " ";
            updateComposition();
        } else if (e.key === 'Enter') {
            finalizeSyllable();
            updateComposition();
        }
    };

    const handleKeyup = (e: KeyboardEvent) => {
        shiftHeld = e.shiftKey;
        pressedCodes.delete(e.code);
        pressedCodes = pressedCodes;
    };

    const resetAll = () => {
        finalizedText = "";
        activeSyllable = "";
        composition = { onset: [], vowel: [], coda: [] };
        startTime = null;
        keystrokes = 0;
        correctKeystrokes = 0;
        wpm = 0;
        accuracy = 100;
        combo = 0;
        isFinished = false;
        currentKeyIdx = Math.floor(Math.random() * (filteredKeys().length || 1));
    };

    const nextSentence = () => {
        resetAll();
        currentSentenceIdx = (currentSentenceIdx + 1) % (texts as Record<string, string[]>)[selectedTextKey].length;
    };

    const currentSebeol = $derived(currentTargetChar ? getSebeolsikKeystrokes(currentTargetChar, layouts[selectedLayout]).map(k => k.replace('Key', '').replace('Digit', '')).join(' ') : '');
    const currentDubeol = $derived(currentTargetChar ? getDubeolsikKeystrokes(currentTargetChar) : '');

    $effect(() => {
        userInput;
        updateMetrics();
    });
</script>

<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />

<div class="fixed inset-0 z-[100] bg-[#0a0a0c] flex flex-col items-center justify-center p-12 text-center sm:hidden">
    <div class="space-y-4 animate-in fade-in zoom-in duration-500">
        <div class="text-4xl mb-6">⚠️</div>
        <h1 class="text-xl font-bold text-white">모바일에서 지원하지 않습니다</h1>
        <p class="text-sm text-[#666666] leading-relaxed">
            본 서비스는 정교한 키보드 연습을 위해<br/>
            PC 환경에 최적화되어 있습니다.
        </p>
    </div>
</div>

<div class="w-full flex flex-col items-center justify-center space-y-8 select-none h-full pt-12">
    {#if showMore || showTextSelect}
        <div class="fixed inset-0 z-50 cursor-default" role="presentation" onclick={() => { showMore = false; showTextSelect = false; }}></div>
    {/if}

    <div class="w-full max-w-4xl flex flex-col sm:flex-row justify-between items-center text-[14px] sm:text-[15px] text-[#666666] mb-4 gap-4 relative">
        <div class="flex items-center gap-4">
            {#if appMode === 'sentence'}
                <button onclick={nextSentence} class="hover:text-white transition-colors">
                    [넘기기]
                </button>
                <span class="opacity-50">#{currentSentenceIdx + 1}</span>
            {/if}
            <button onclick={() => { selectedLayout = selectedLayout === '390' ? 'Final' : '390'; resetAll(); }} 
                    class="hover:text-white transition-colors">
                [배열: {selectedLayout === '390' ? '390' : '최종'}]
            </button>
            <button onclick={() => { appMode = appMode === 'sentence' ? 'key' : 'sentence'; resetAll(); }}
                    class="hover:text-white transition-colors"
                    title="Tab 키로 변경 가능">
                [모드: {appMode === 'sentence' ? '문장' : '낱자'}]
            </button>
            
            {#if appMode === 'sentence'}
                <div class="relative">
                    <button 
                        onclick={() => { showTextSelect = !showTextSelect; showMore = false; }}
                        class="flex items-center gap-2 hover:text-white transition-colors group">
                        <span>[{selectedTextKey}]</span>
                        <span class="text-[11px] opacity-30 group-hover:opacity-100 transition-opacity">▼</span>
                    </button>
                    
                    <Dropdown show={showTextSelect} class="min-w-[280px] p-0 overflow-hidden">
                        <div class="p-3 border-b border-white/5 bg-white/[0.02]">
                            <input 
                                type="text" 
                                bind:value={searchText}
                                placeholder="연습할 글귀 검색..."
                                class="w-full bg-white/5 border border-white/10 rounded-lg px-3 py-2 text-[13px] outline-none focus:border-white/20 transition-colors"
                            />
                        </div>
                        <div class="flex flex-col gap-1 p-2 max-h-[320px] overflow-y-auto">
                            {#each Object.keys(texts).filter(t => t.toLowerCase().includes(searchText.toLowerCase()) && compatibleTexts[t]) as title}
                                <button 
                                    onclick={() => { selectedTextKey = title; showTextSelect = false; searchText = ""; resetAll(); }}
                                    class="w-full text-left px-4 py-3 rounded-lg text-[14px] transition-all {selectedTextKey === title ? 'bg-white/10 text-white' : 'text-[#666666] hover:bg-white/5 hover:text-white'}">
                                    {title}
                                </button>
                            {:else}
                                <div class="px-4 py-8 text-center text-[#444444] text-xs">
                                    검색 결과가 없습니다
                                </div>
                            {/each}
                        </div>
                    </Dropdown>
                </div>
            {/if}
            
            <div class="relative">
                <button onclick={() => { showMore = !showMore; showTextSelect = false; }} class="hover:text-white transition-colors">
                    [더보기]
                </button>
                
                <Dropdown show={showMore} class="w-56 p-5">
                    <div class="flex flex-col gap-5">
                        <label class="flex items-center justify-between cursor-pointer group text-[14px]">
                            <span class="group-hover:text-white transition-colors">가상 키보드</span>
                            <input type="checkbox" bind:checked={showKeyboard} class="accent-brand-primary" />
                        </label>
                        <label class="flex items-center justify-between cursor-pointer group text-[14px]">
                            <span class="group-hover:text-white transition-colors">입력 힌트</span>
                            <input type="checkbox" bind:checked={showHints} class="accent-brand-primary" />
                        </label>
                        <hr class="border-white/5" />
                        <div class="space-y-2 opacity-40">
                            <p class="text-sm font-bold">메모</p>
                            <p class="text-sm leading-relaxed">1. 이 페이지는 리디주식회사에서 제공한 리디바탕 글꼴이 사용되어 있습니다.</p>
                            <p class="text-sm leading-relaxed">2. 개발자의 더 다양한 프로젝트 및 정보를 확인하시려면 <a href="https://pf.ai.ai.kr" target="_blank" rel="noopener noreferrer" class="text-brand-primary hover:text-white transition-colors">https://pf.ai.ai.kr</a>을 확인해주세요.</p>
                        </div>
                    </div>
                </Dropdown>
            </div>

            {#if appMode === 'key'}
                <div class="flex items-center gap-2 ml-4 px-2 py-1 bg-white/5 rounded border border-white/5">
                    {#each lessons as lesson}
                        <button onclick={() => { selectedLesson = lesson.id; resetAll(); }}
                                class="px-2 py-0.5 text-[10px] rounded transition-all {selectedLesson === lesson.id ? 'bg-[#333333] text-white' : 'text-[#666666] hover:text-white'}">
                            {lesson.label}
                        </button>
                    {/each}
                </div>
            {/if}
        </div>

        <div class="flex items-center gap-12">
            <div class="flex flex-col items-center">
                <span class="metric-label">속도</span>
                <span class="metric-value">{wpm}</span>
            </div>
            <div class="flex flex-col items-center">
                <span class="metric-label">정확도</span>
                <span class="metric-value">{accuracy}%</span>
            </div>
            <div class="flex flex-col items-center">
                <span class="metric-label">콤보</span>
                <span class="metric-value">{combo}</span>
            </div>
        </div>
    </div>

    <div class="w-full max-w-5xl mx-auto flex flex-col justify-center px-4 sm:px-0 py-12">
        {#if appMode === 'sentence'}
            <div class="text-2xl sm:text-3xl lg:text-4xl font-medium leading-[2.2] tracking-widest break-keep text-center font-['Ridibatang',serif]">
                {#each currentText.split("") as char, i}
                    {@const isFinalized = i < finalizedText.length}
                    {@const isCurrent = i === finalizedText.length}
                    {@const isError = isFinalized && finalizedText[i] !== char}
                    {@const isSpace = char === ' '}
                    
                    <span class="relative inline-block mx-[2.5px] transition-all duration-200 ease-out rounded-sm {isError ? 'text-red-500 bg-red-500/5' : isFinalized ? 'text-white/50' : isCurrent && !isSpace && !activeSyllable ? 'text-white/30' : 'text-white/10'}" 
                          style="min-width: {isSpace ? '0.7em' : 'auto'}; transform: {isCurrent && activeSyllable ? 'scale(1.1)' : 'scale(1)'};">
                        {isSpace ? '\u00A0' : char}
                        {#if isCurrent && activeSyllable}
                            <span class="absolute inset-0 flex items-center justify-center z-10 text-white font-medium" in:fade={{ duration: 100 }}>{activeSyllable}</span>
                        {/if}
                        {#if isCurrent}
                            <div class="absolute -bottom-2 left-0 w-full h-[1.5px] bg-white/40 rounded-full transition-all duration-300"></div>
                        {/if}
                    </span>
                {/each}
            </div>

            {#if showHints}
                <div class="mt-12 h-10 flex flex-col items-center justify-center" style="opacity: {currentTargetChar && !isFinished ? 1 : 0};">
                    {#if currentTargetChar === ' '}
                        <p class="metric-label">[space]</p>
                    {:else if currentTargetChar}
                        <div class="flex items-center gap-6 text-[13px] tracking-wide text-white/20 font-medium">
                            <span class="flex items-center gap-2">세벌: <span class="text-white/60">{currentSebeol}</span></span>
                            <span class="w-[1px] h-3 bg-white/5"></span>
                            <span class="flex items-center gap-2">두벌: <span class="text-white/60">{currentDubeol}</span></span>
                        </div>
                    {/if}
                </div>
            {/if}
        {:else}
            <div class="flex flex-col items-center justify-center gap-4 animate-in fade-in duration-500">
                <div class="text-[6rem] sm:text-[10rem] font-bold text-white tracking-tighter h-48 flex items-center justify-center">
                    {currentTargetKey.char}
                </div>
                <div class="text-[#666666] text-sm tracking-widest uppercase">{currentTargetKey.desc}</div>
            </div>
        {/if}
    </div>
    
    {#if showKeyboard}
        <div class="mt-4 flex justify-center w-full transform scale-[0.65] sm:scale-75 md:scale-90 lg:scale-100 transition-all duration-500 origin-top z-20" transition:fade={{ duration: 300 }}>
            <Keyboard layoutName={selectedLayout} pressedKeys={pressedKeysObj} isShiftPressed={isShiftPressed} activeKey={appMode === 'key' ? currentTargetKey.code : ''} />
        </div>
    {/if}
</div>
