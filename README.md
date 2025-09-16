
  const onDrop = useCallback((acceptedFiles: File[]) => {
    // 2. onDropではstateを更新するだけにする
    if (acceptedFiles.length > 0) {
      setDroppedFiles(acceptedFiles);
    }
  }, []);

  // 3. droppedFilesの変更を監視し、ブラウザ環境でイベントを発火させる
  useEffect(() => {
    // droppedFilesがnullでない、かつinputRefが存在する場合のみ実行
    if (droppedFiles && inputRef?.current) {
      console.log('useEffect: ブラウザでイベントを発火させます。');
      
      // このコードブロックは常にブラウザで実行されるため安全
      const dataTransfer = new DataTransfer();
      droppedFiles.forEach((file) => dataTransfer.items.add(file));
      inputRef.current.files = dataTransfer.files;

      const event = new Event('change', { bubbles: true });
      inputRef.current.dispatchEvent(event);

      // 処理後にstateをリセット
      setDroppedFiles(null);
    }
  }, [droppedFiles, inputRef]); // 依存配列にdroppedFilesとinputRefを指定
