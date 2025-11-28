#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <errno.h>

#define MAX_STUDENTS 33000
#define NAME_LEN 64
#define NUM_TRIALS 40   // 임의 연산 횟수

typedef struct {
    int id;
    char name[NAME_LEN];
    char gender;            // 'M' or 'F'
    int korean;
    int english;
    int math;
    int mul_of_score;       // korean * english * math
} Student;

// ------------------- AVL 트리 구조체 -------------------
typedef struct AVLNode {
    Student data;
    int height;
    struct AVLNode* left;
    struct AVLNode* right;
} AVLNode;

// ---- 전역 변수: 정렬 시 비교 횟수 카운트 및 AVL 루트 ----
long long g_sort_comp_count = 0;
AVLNode* g_avl_root = NULL;

/* =====================================================
 * AVL 트리 유틸리티 함수
 * ===================================================== */

int max(int a, int b) {
    return (a > b) ? a : b;
}

int getHeight(AVLNode* node) {
    return (node == NULL) ? 0 : node->height;
}

// 균형 인수(Balance Factor) 계산
int getBalance(AVLNode* node) {
    return (node == NULL) ? 0 : getHeight(node->left) - getHeight(node->right);
}

// 새 노드 생성
AVLNode* createNode(Student s) {
    AVLNode* newNode = (AVLNode*)malloc(sizeof(AVLNode));
    if (newNode == NULL) {
        perror("메모리 할당 실패");
        exit(EXIT_FAILURE);
    }
    newNode->data = s;
    newNode->height = 1; // 새 노드는 높이 1
    newNode->left = NULL;
    newNode->right = NULL;
    return newNode;
}

// 오른쪽 회전 (Right Rotation)
AVLNode* rightRotate(AVLNode* y) {
    AVLNode* x = y->left;
    AVLNode* T2 = x->right;

    // 회전 수행
    x->right = y;
    y->left = T2;

    // 높이 업데이트
    y->height = max(getHeight(y->left), getHeight(y->right)) + 1;
    x->height = max(getHeight(x->left), getHeight(x->right)) + 1;

    return x; // 새로운 루트
}

// 왼쪽 회전 (Left Rotation)
AVLNode* leftRotate(AVLNode* x) {
    AVLNode* y = x->right;
    AVLNode* T2 = y->left;

    // 회전 수행
    y->left = x;
    x->right = T2;

    // 높이 업데이트
    x->height = max(getHeight(x->left), getHeight(x->right)) + 1;
    y->height = max(getHeight(y->left), getHeight(y->right)) + 1;

    return y; // 새로운 루트
}


/* =====================================================
 * AVL 트리 탐색, 삽입, 삭제 함수
 * ===================================================== */

 // AVL 탐색: MUL_OF_SCORE로 key를 찾는다.
AVLNode* avl_search(AVLNode* root, int key, int* compCount) {
    if (root == NULL) return NULL;

    (*compCount)++;
    if (key == root->data.mul_of_score) return root;

    if (key < root->data.mul_of_score) {
        return avl_search(root->left, key, compCount);
    }
    else {
        return avl_search(root->right, key, compCount);
    }
}


// AVL 삽입: 재귀적으로 삽입하고, 균형을 맞춘다.
// compCount에는 비교 횟수만 카운트하며, 균형 잡기(회전)에 필요한 비교는 제외합니다.
AVLNode* avl_insert(AVLNode* node, int key, int* n, int* compCount, int* last_id) {
    // 1. 일반 BST 삽입 (종료 조건)
    if (node == NULL) {
        // 새 ID/학생 생성
        (*last_id)++;
        Student s;
        s.id = *last_id;
        s.gender = (rand() % 2 == 0) ? 'F' : 'M';
        s.korean = rand() % 101;
        s.english = rand() % 101;
        s.math = rand() % 101;
        s.mul_of_score = key;
        strcpy_s(s.name, sizeof(s.name), "DUMMY");
        (*n)++;
        return createNode(s);
    }

    (*compCount)++; // 데이터 비교 횟수 카운트
    if (key < node->data.mul_of_score) {
        node->left = avl_insert(node->left, key, n, compCount, last_id);
    }
    else if (key > node->data.mul_of_score) {
        node->right = avl_insert(node->right, key, n, compCount, last_id);
    }
    else {
        // 중복이면 삽입 실패 (학생 수 카운트 안 함)
        return node;
    }

    // 2. 현재 노드의 높이 업데이트
    node->height = 1 + max(getHeight(node->left), getHeight(node->right));

    // 3. 균형 인수 확인 및 불균형 처리 (회전)
    int balance = getBalance(node);

    // LL Case
    if (balance > 1 && key < node->left->data.mul_of_score) {
        return rightRotate(node);
    }
    // RR Case
    if (balance < -1 && key > node->right->data.mul_of_score) {
        return leftRotate(node);
    }
    // LR Case
    if (balance > 1 && key > node->left->data.mul_of_score) {
        node->left = leftRotate(node->left);
        return rightRotate(node);
    }
    // RL Case
    if (balance < -1 && key < node->right->data.mul_of_score) {
        node->right = rightRotate(node->right);
        return leftRotate(node);
    }

    return node;
}

// AVL 삭제 시 사용할 최소값 노드 찾기
AVLNode* findMinNode(AVLNode* node) {
    AVLNode* current = node;
    while (current->left != NULL) {
        current = current->left;
    }
    return current;
}

// AVL 삭제: 재귀적으로 삭제하고, 균형을 맞춘다.
AVLNode* avl_delete(AVLNode* root, int key, int* n, int* compCount) {
    if (root == NULL) return root;

    (*compCount)++; // 데이터 비교 횟수 카운트
    if (key < root->data.mul_of_score) {
        root->left = avl_delete(root->left, key, n, compCount);
    }
    else if (key > root->data.mul_of_score) {
        root->right = avl_delete(root->right, key, n, compCount);
    }
    else {
        // 삭제할 노드 발견
        AVLNode* temp = NULL;

        if ((root->left == NULL) || (root->right == NULL)) {
            temp = root->left ? root->left : root->right;

            if (temp == NULL) { // 리프 노드
                temp = root;
                root = NULL;
            }
            else { // 자식이 하나
                // temp의 데이터를 root로 복사하고 temp 메모리 해제
                root->data = temp->data;
                root->left = root->right = NULL;
            }
            free(temp);
            (*n)--; // 학생 수 감소
        }
        else {
            // 자식이 둘: 후임자(Inorder Successor)를 찾아 데이터 복사 후 재귀 삭제
            temp = findMinNode(root->right);

            // 후임자의 데이터로 현재 노드 덮어쓰기
            root->data = temp->data;

            // 후임자를 오른쪽 서브트리에서 재귀적으로 삭제
            root->right = avl_delete(root->right, temp->data.mul_of_score, n, compCount);
            // 재귀 호출에서 이미 학생 수 감소가 처리되었으므로 여기서 n--를 다시 하지 않습니다.
        }
    }

    if (root == NULL) return root;

    // 2. 높이 업데이트
    root->height = 1 + max(getHeight(root->left), getHeight(root->right));

    // 3. 균형 확인 및 불균형 처리 (회전)
    int balance = getBalance(root);

    // LL Case
    if (balance > 1 && getBalance(root->left) >= 0)
        return rightRotate(root);

    // LR Case
    if (balance > 1 && getBalance(root->left) < 0) {
        root->left = leftRotate(root->left);
        return rightRotate(root);
    }

    // RR Case
    if (balance < -1 && getBalance(root->right) <= 0)
        return leftRotate(root);

    // RL Case
    if (balance < -1 && getBalance(root->right) > 0) {
        root->right = rightRotate(root->right);
        return leftRotate(root);
    }

    return root;
}


/* =====================================================
 * 기본 탐색 함수 (기존 코드)
 * ===================================================== */

 // 순차탐색: MUL_OF_SCORE 필드에서 key를 찾는다.
int sequential_search(Student arr[], int n, int key, int* compCount) {
    int i;
    *compCount = 0;
    for (i = 0; i < n; i++) {
        (*compCount)++;
        if (arr[i].mul_of_score == key) {
            return i;
        }
    }
    return -1;
}

// 이진탐색: 정렬된 배열에서 MUL_OF_SCORE로 key를 찾는다.
int binary_search(Student arr[], int n, int key, int* compCount) {
    int low = 0, high = n - 1;
    *compCount = 0;

    while (low <= high) {
        int mid = (low + high) / 2;
        (*compCount)++;

        if (arr[mid].mul_of_score == key) {
            return mid;
        }
        else if (key < arr[mid].mul_of_score) {
            high = mid - 1;
        }
        else {
            low = mid + 1;
        }
    }
    return -1;
}

/* =====================================================
 * 정렬용 비교 함수(qsort) (기존 코드)
 * ===================================================== */
int compare_by_mul(const void* a, const void* b) {
    const Student* s1 = (const Student*)a;
    const Student* s2 = (const Student*)b;

    g_sort_comp_count++;

    if (s1->mul_of_score < s2->mul_of_score) return -1;
    else if (s1->mul_of_score > s2->mul_of_score) return 1;
    else {
        if (s1->id < s2->id) return -1;
        else if (s1->id > s2->id) return 1;
        else return 0;
    }
}

/* =====================================================
 * 정렬되지 않은 배열: 삽입/삭제 (기존 코드)
 * ===================================================== */

int last_id;
int unsorted_insert(Student arr[], int* n, int key, int* compCount, int* last_id)
{
    // key 값이 mul_of_score가 동일한 학생이 있는지 검사
    int idx = sequential_search(arr, *n, key, compCount);
    if (idx != -1) {
        return 0; // 이미 존재
    }

    if (*n >= MAX_STUDENTS) {
        return 0; // 공간 부족
    }

    // ★ 새 ID 생성: 마지막 ID에서 1 증가
    (*last_id)++;
    int new_id = *last_id;

    // ★ gender 난수 생성
    char gender = (rand() % 2 == 0) ? 'F' : 'M';

    // ★ 과목 점수 난수 생성
    int ko = rand() % 101;
    int en = rand() % 101;
    int ma = rand() % 101;

    // ★ 학생 생성
    Student s;
    s.id = new_id;
    s.gender = gender;
    s.korean = ko;
    s.english = en;
    s.math = ma;
    s.mul_of_score = key; // key로 고정 (기존 코드에서 난수 대신 key 사용)
    strcpy_s(s.name, sizeof(s.name), "RANDOM");

    // 배열에 삽입
    arr[*n] = s;
    (*n)++;

    return 1;
}


// 삭제(정렬X): 순차탐색으로 찾고, 찾으면 마지막 원소로 덮어써서 삭제
int unsorted_delete(Student arr[], int* n, int key, int* compCount) {
    int idx = sequential_search(arr, *n, key, compCount);
    if (idx == -1) return 0; // 없음

    arr[idx] = arr[*n - 1];
    (*n)--;
    return 1;
}

/* =====================================================
 * 정렬된 배열: 삽입/삭제 (기존 코드)
 * ===================================================== */

 // 정렬 배열에서 key가 들어갈 첫 위치(lower_bound) 찾기
int lower_bound(Student arr[], int n, int key, int* compCount) {
    int low = 0, high = n; // high는 n까지
    *compCount = 0;

    while (low < high) {
        int mid = (low + high) / 2;
        (*compCount)++;
        if (arr[mid].mul_of_score < key)
            low = mid + 1;
        else
            high = mid;
    }
    return low;
}

// 삽입(정렬O): lower_bound로 위치 찾고 shift 후 삽입
int sorted_insert(Student arr[], int* n, int key, int* compCount) {
    if (*n >= MAX_STUDENTS) return 0;

    // 1) lower_bound로 삽입 위치 찾기
    int pos = lower_bound(arr, *n, key, compCount);
    // compCount에는 lower_bound에서의 "비교 횟수"가 들어있음

    // 2) 중복 체크 (이 비교도 데이터 비교이므로 1회 카운트)
    if (pos < *n) {
        (*compCount)++;  // arr[pos].mul_of_score == key 비교
        if (arr[pos].mul_of_score == key) {
            return 0;      // 중복이면 삽입 안 함
        }
    }

    // 3) 오른쪽 shift 수행 + 이동 횟수 카운트
    // 밀리는 원소 개수는 (*n - pos)
    int moves = (*n - pos);

    for (int i = *n; i > pos; i--) {
        arr[i] = arr[i - 1];
    }

    // shift 이동 횟수를 비교횟수에 포함 (데이터 이동도 비용으로 간주)
    *compCount += moves;

    // 4) 새 학생(더미) 삽입
    Student dummy;
    dummy.id = 0;
    dummy.gender = 'M';
    dummy.korean = dummy.english = dummy.math = 0;
    dummy.mul_of_score = key;
    strcpy_s(dummy.name, sizeof(dummy.name), "DUMMY");

    arr[pos] = dummy;
    (*n)++;

    return 1;
}


// 삭제(정렬O): binary_search로 찾고 shift로 삭제
int sorted_delete(Student arr[], int* n, int key, int* compCount) {
    int idx = binary_search(arr, *n, key, compCount);
    if (idx == -1) return 0;

    // 왼쪽 shift 수행 + 이동 횟수 카운트
    int moves = (*n - 1 - idx);

    for (int i = idx; i < *n - 1; i++) {
        arr[i] = arr[i + 1];
    }
    (*n)--;

    // shift 이동 횟수를 비교횟수에 포함 (데이터 이동도 비용으로 간주)
    *compCount += moves;

    return 1;
}


/* =====================================================
 * CSV 파싱 (기존 코드)
 * ===================================================== */
int parse_student_line(const char* line, Student* stu) {
    char buffer[256];
    errno_t err;
    char* token;
    char* context = NULL;

    err = strcpy_s(buffer, sizeof(buffer), line);
    if (err != 0) {
        fprintf(stderr, "경고: 한 줄이 너무 길어 잘렸을 수 있습니다.\n");
    }

    token = strtok_s(buffer, ",\n", &context);
    if (!token) return 0;
    stu->id = atoi(token);
    last_id = stu->id;

    token = strtok_s(NULL, ",\n", &context);
    if (!token) return 0;
    err = strcpy_s(stu->name, sizeof(stu->name), token);

    token = strtok_s(NULL, ",\n", &context);
    if (!token) return 0;
    stu->gender = token[0];

    token = strtok_s(NULL, ",\n", &context);
    if (!token) return 0;
    stu->korean = atoi(token);

    token = strtok_s(NULL, ",\n", &context);
    if (!token) return 0;
    stu->english = atoi(token);

    token = strtok_s(NULL, ",\n", &context);
    if (!token) return 0;
    stu->math = atoi(token);

    stu->mul_of_score = stu->korean * stu->english * stu->math;

    return 1;
}

/* =====================================================
 * 전역 배열 (기존 코드)
 * ===================================================== */
Student students[MAX_STUDENTS];
Student sorted_students[MAX_STUDENTS];

int main(void) {
    FILE* fp = NULL;
    errno_t err;
    char line[256];
    int n = 0;
    // last_id는 전역 변수로 이미 선언됨

    // 1. 파일 열기
    err = fopen_s(&fp, "students.csv", "r");
    if (err != 0 || fp == NULL) {
        fprintf(stderr, "파일 열기 실패: students.csv (err=%d). \n파일이 같은 경로에 있는지 확인하세요.\n", (int)err);
        return 1;
    }

    // 2. 헤더 버리기
    if (!fgets(line, sizeof(line), fp)) {
        fprintf(stderr, "파일이 비어있습니다.\n");
        fclose(fp);
        return 1;
    }

    // 3. 학생 읽기
    while (fgets(line, sizeof(line), fp)) {
        if (n >= MAX_STUDENTS) break;
        if (line[0] == '\n' || line[0] == '\r') continue;

        if (parse_student_line(line, &students[n])) n++;
    }
    fclose(fp);

    if (n == 0) {
        fprintf(stderr, "학생 데이터가 없습니다.\n");
        return 1;
    }
    printf("읽은 학생 수: %d명\n", n);

    // sorted_students 초기화 + 정렬 1회
    for (int i = 0; i < n; i++) {
        sorted_students[i] = students[i];
    }
    g_sort_comp_count = 0;
    qsort(sorted_students, n, sizeof(Student), compare_by_mul);
    printf("\n[정렬 1회 수행 완료] 정렬 비교 횟수: %lld\n", g_sort_comp_count);

    // 난수 초기화
    srand((unsigned)time(NULL));

    // 세 실험에서 동일한 key/연산을 쓰고 싶어서 미리 생성
    int keys[NUM_TRIALS];
    for (int i = 0; i < NUM_TRIALS; i++) {
        keys[i] = rand() % 1000001;
    }

    /* =====================================================
     * A. 정렬되지 않은 배열 실험 (순차 탐색)
     * ===================================================== */
    printf("\n==============================\n");
    printf("A) 정렬되지 않은 배열 (순차탐색 기반)\n");
    printf("==============================\n");
    printf("시도, key, op(0=ins,1=del,2=seq), 이번비교, 누적비교, 평균비교, 결과\n");

    int n_uns = n;
    long long uns_total_comp = 0;
    int uns_last_id = last_id;

    for (int t = 0; t < NUM_TRIALS; t++) {
        int key = keys[t];
        int op = key % 3;
        int comp = 0;
        int ok = 0;

        if (op == 0) {
            ok = unsorted_insert(students, &n_uns, key, &comp, &uns_last_id);
        }
        else if (op == 1) {
            ok = unsorted_delete(students, &n_uns, key, &comp);
        }
        else {
            (void)sequential_search(students, n_uns, key, &comp);
            ok = 1;
        }

        uns_total_comp += comp;
        double avg = (double)uns_total_comp / (t + 1);

        printf("%d, %d, %d, %d, %lld, %.2f, %s\n",
            t + 1, key, op, comp, uns_total_comp, avg,
            ok ? "OK" : "FAIL");
    }

    /* =====================================================
     * B. 정렬된 배열 실험 (이진 탐색)
     * ===================================================== */
    printf("\n==============================\n");
    printf("B) 정렬된 배열 (이진탐색 기반)\n");
    printf("==============================\n");
    printf("시도, key, op(0=ins,1=del,2=bin), 이번비교, 누적비교, 평균비교, 결과\n");

    int n_sort = n;
    long long sort_total_comp = 0;

    for (int t = 0; t < NUM_TRIALS; t++) {
        int key = keys[t];
        int op = key % 3;
        int comp = 0;
        int ok = 0;

        if (op == 0) {
            ok = sorted_insert(sorted_students, &n_sort, key, &comp);
        }
        else if (op == 1) {
            ok = sorted_delete(sorted_students, &n_sort, key, &comp);
        }
        else {
            (void)binary_search(sorted_students, n_sort, key, &comp);
            ok = 1;
        }

        sort_total_comp += comp;
        double avg = (double)sort_total_comp / (t + 1);

        printf("%d, %d, %d, %d, %lld, %.2f, %s\n",
            t + 1, key, op, comp, sort_total_comp, avg,
            ok ? "OK" : "FAIL");
    }

    /* =====================================================
     * C. AVL 트리 실험 (O(log N) 보장)
     * ===================================================== */
    printf("\n==============================\n");
    printf("C) AVL 트리 (O(log N) 보장)\n");
    printf("==============================\n");
    printf("시도, key, op(0=ins,1=del,2=search), 이번비교, 누적비교, 평균비교, 결과\n");

    int n_avl = 0; // AVL의 초기 학생 수는 0으로 시작
    long long avl_total_comp = 0;
    int avl_last_id = last_id;

    // 초기 배열 데이터로 AVL 트리를 구성 (성능 측정 제외)
    for (int i = 0; i < n; i++) {
        int comp = 0;
        g_avl_root = avl_insert(g_avl_root, students[i].mul_of_score, &n_avl, &comp, &avl_last_id);
    }
    printf("(초기 데이터 %d개 로드 완료. 비교 횟수 측정 시작)\n", n_avl);


    for (int t = 0; t < NUM_TRIALS; t++) {
        int key = keys[t];
        int op = key % 3;
        int comp = 0;
        int ok = 0;
        int initial_n_avl = n_avl;

        if (op == 0) {
            g_avl_root = avl_insert(g_avl_root, key, &n_avl, &comp, &avl_last_id);
            ok = (n_avl > initial_n_avl); // 학생 수가 증가했으면 삽입 성공
        }
        else if (op == 1) {
            g_avl_root = avl_delete(g_avl_root, key, &n_avl, &comp);
            ok = (n_avl < initial_n_avl); // 학생 수가 감소했으면 삭제 성공
        }
        else {
            AVLNode* result = avl_search(g_avl_root, key, &comp);
            ok = (result != NULL);
        }

        avl_total_comp += comp;
        double avg = (double)avl_total_comp / (t + 1);

        printf("%d, %d, %d, %d, %lld, %.2f, %s\n",
            t + 1, key, op, comp, avl_total_comp, avg,
            ok ? "OK" : "FAIL");
    }

    return 0;
}
