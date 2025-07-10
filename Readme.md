import { fetchRepositoryContentByDocumentationId } from '@/services/documentation';
import { AppDispatch, RootState } from '@/store/store';
import { PayloadAction, createAsyncThunk, createSlice } from '@reduxjs/toolkit';
import { AxiosInstance } from 'axios';

import { ChangeStatus, FileType, getChange } from '@/lib/indexed-db';
import { findNodeRecursive } from '@/lib/tree-mutator';

// --- STATE SHAPE ---

export interface EditorFileState {
  path: string;
  baseSha: string | null; // The COMMIT SHA this file's content is based on.
  baseContentSha: string | null; // The BLOB SHA (uniqueId) of the original content.
  originalContent: string | null;
  currentContent: string | null;
  isDirty: boolean;
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error?: string | null;
  fileType: FileType | null;
}

export interface EditorState {
  files: Record<string, EditorFileState>; // A dictionary of all open files, keyed by path
  activeFile: string | null; // The path of the file currently visible in the editor
}

export const initialState: EditorState = {
  files: {},
  activeFile: null,
};

// --- ASYNC THUNKS ---

export const openFile = createAsyncThunk<
  EditorFileState,
  { path: string; api: AxiosInstance },
  { state: RootState; dispatch: AppDispatch }
>('editor/openFile', async ({ path, api }, { getState, rejectWithValue }) => {
  try {
    const state = getState();
    const { activeBranch, baseSha: workspaceCommitSha, tree } = state.fileTree;
    const documentationId = state.selectedDocumentation?.id;
    const organizationId = state.selectedOrg?.id;

    if (!activeBranch) throw new Error('Cannot open file without an active branch.');
    if (!documentationId || !organizationId)
      throw new Error('Documentation and Organization must be selected.');

    // 1. Check for an unsaved version in IndexedDB first.
    const unsavedVersion = await getChange(activeBranch, path);
    if (unsavedVersion) {
      return {
        path: path,
        baseSha: unsavedVersion.baseSha,
        baseContentSha: unsavedVersion.baseContentSha,
        originalContent: unsavedVersion.originalContent as string, // Assuming text
        currentContent: unsavedVersion.currentContent as string,
        isDirty: (unsavedVersion.originalContent ?? '') !== (unsavedVersion.currentContent ?? ''),
        status: 'succeeded',
        fileType: unsavedVersion.fileType,
      };
    }

    // 2. If no draft, fetch the clean file from the backend.
    const fileNode = findNodeRecursive(tree, path);
    if (!fileNode || fileNode.type !== 'blob') {
      throw new Error('File not found in the tree or is a directory.');
    }

    if (typeof workspaceCommitSha !== 'string') {
      throw new Error('Cannot fetch file content without a valid commit SHA.');
    }

    const fileContent = await fetchRepositoryContentByDocumentationId(
      api,
      organizationId,
      documentationId,
      path,
      workspaceCommitSha // Using the commit SHA as the ref
    );

    return {
      path: path,
      baseSha: workspaceCommitSha,
      baseContentSha: fileNode.uniqueId, // Using uniqueId as the blob SHA
      originalContent: fileContent,
      currentContent: fileContent,
      isDirty: false,
      status: 'succeeded',
      fileType: FileType.Text, // Assuming text, can be enhanced
    };
  } catch (error) {
    console.error('Failed to open file:', error);
    return rejectWithValue(error);
  }
});

// --- SLICE DEFINITION ---

const editorSlice = createSlice({
  name: 'editor',
  initialState,
  reducers: {
    updateFileContent: (state, action: PayloadAction<{ path: string; newContent: string }>) => {
      const { path, newContent } = action.payload;
      const file = state.files[path];
      if (file) {
        file.currentContent = newContent;
        file.isDirty = file.currentContent !== file.originalContent;
      }
    },
    setActiveFile: (state, action: PayloadAction<string | null>) => {
      state.activeFile = action.payload;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(openFile.pending, (state, action) => {
        const { path } = action.meta.arg;
        if (!state.files[path]) {
          state.files[path] = {
            path,
            status: 'loading',
            baseSha: null,
            baseContentSha: null,
            originalContent: null,
            currentContent: null,
            isDirty: false,
            fileType: null,
          };
        } else {
          state.files[path].status = 'loading';
        }
      })
      .addCase(openFile.fulfilled, (state, action) => {
        const fileState = action.payload;
        state.files[fileState.path] = fileState;
        state.activeFile = fileState.path;
      })
      .addCase(openFile.rejected, (state, action) => {
        const { path } = action.meta.arg;
        if (state.files[path]) {
          state.files[path].status = 'failed';
          state.files[path].error = action.error.message;
        }
      });
  },
});

export const { updateFileContent, setActiveFile } = editorSlice.actions;

export default editorSlice.reducer;
